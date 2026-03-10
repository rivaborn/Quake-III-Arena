# code/q3_ui/ui_sppostgame.c — Enhanced Analysis

## Architectural Role
This file implements the postgame UI sequence for single-player campaigns in the `q3_ui` (base-Q3A) QVM menu module. It bridges the **game VM** (which signals match end via console command) to the **client UI framework** (which manages menu stack and drawing). The postgame menu orchestrates a timed three-phase presentation—podium rankings → award medals → interactive navigation buttons—and integrates with the tier-based campaign progression system by computing valid next levels and triggering tier cinematics upon completion.

## Key Cross-References

### Incoming (who depends on this file)
- **q3_ui module's `vmMain` dispatch** (in other files like `ui_main.c`): routes console command `sppostgame` to `UI_SPPostgameMenu_f`
- **Game VM** (e.g., `code/game/g_main.c`): sends postgame data via `trap_Cmd_ExecuteText("sppostgame ...")` at match end
- **Menu framework** (`ui_qmenu.c`): calls registered callback pointers (`menu.draw`, `menu.key`) each frame to drive state progression
- **Client engine** (cgame VM): reads final scores/ranks from configstrings before match end; postgame menu visualizes this data

### Outgoing (what this file depends on)
- **UI subsystem functions** (defined in other `q3_ui/*.c` files):
  - `UI_GetArenaInfoByMap`, `UI_GetArenaInfoByNumber`, `UI_SPArena_Start` — query/navigate level/tier database
  - `UI_TierCompleted`, `UI_ShowTierVideo`, `UI_GetNumSPTiers`, `UI_GetSpecialArenaInfo` — tier progression logic
  - `UI_SetBestScore`, `UI_LogAwardData`, `UI_GetAwardLevel` — persist scores and award stats
  - `UI_GetCurrentGame`, `UI_PopMenu`, `UI_PushMenu` — menu stack management
  - Draw utilities: `UI_DrawProportionalString`, `UI_DrawNamedPic`, `UI_DrawString`, `UI_ProportionalStringWidth`
  - `Menu_*` (menu framework): `Menu_AddItem`, `Menu_Draw`, `Menu_DefaultKey`
- **Engine syscalls** (via `trap_*` indirection into qcommon):
  - `trap_GetConfigString` — read player info, server ID for validation
  - `trap_Cvar_Set`, `trap_Cvar_SetValue`, `trap_Cvar_VariableValue` — read/write cvars (`ui_spScoreboard`, `nextmap`, `ui_spSelection`)
  - `trap_S_RegisterSound`, `trap_S_StartLocalSound` — audio assets and playback
  - `trap_R_RegisterShaderNoMip` — register medal/button artwork
  - `trap_Cmd_ExecuteText` — issue commands (`map_restart`, `disconnect`, cinematic playback)
  - `trap_Key_SetCatcher` — manage input focus
- **Shared utilities**:
  - `Info_ValueForKey` — parse configstring key-value pairs
  - `Q_strncpyz`, `Q_CleanStr`, `va`, `Com_sprintf` — string formatting
  - `uis` global (from `ui_local.h`) — UI state (realtime, demoversion, menusp)

## Design Patterns & Rationale

**1. Phase-Based State Machine with Real-Time Progression**
- Phases 1→3 are driven purely by elapsed `uis.realtime` rather than event callbacks. Each frame, `UI_SPPostgameMenu_MenuDraw` compares current time against `postgameMenuInfo.starttime` to determine when to transition.
- **Why**: Avoids tight coupling to a separate "update" tick; the menu is entirely self-contained. UI VMs in Q3 only have a draw callback, no guaranteed update loop, so time-based checks are idiomatic.

**2. Array-Based Award Configuration**
- Six award types are defined in parallel arrays: `ui_medalNames[]`, `ui_medalPicNames[]`, `ui_medalSounds[]`. A numeric enum (`AWARD_ACCURACY` = 0, etc.) indexes these.
- **Why**: Decouples award definition from rendering logic. Adding a new award requires only data changes, not code refactoring.

**3. Lazy Asset Registration via Cache()**
- `UI_SPPostgameMenu_Cache()` pre-registers all shaders and sounds. When `com_buildscript` is true, additional assets are registered to force inclusion in release pak files.
- **Why**: Q3's virtual filesystem and pak bundling require explicit registration to guarantee assets are available and included in deterministic builds. The `Cache()` pattern is standard across all Q3 UI menus.

**4. Validated Server ID Sentinel**
- Stores `postgameMenuInfo.serverId` at init; each draw frame validates it against the current server's `sv_serverid` cvar. Mismatch triggers menu dismissal.
- **Why**: Detects if the server disconnected or a new game started, preventing stale postgame data from corrupting the UI of the next map.

**5. Deferred Button Activation**
- The "Again", "Next", "Menu" buttons are created as inactive (`QMF_INACTIVE` flag) and only activated in phase 3 after awards presentation completes.
- **Why**: Prevents accidental/premature input during animations. Clear phase gates ensure the user experiences the full presentation before interacting.

## Data Flow Through This File

```
[Game VM ends match]
    ↓
[trap_Cmd_ExecuteText("sppostgame <args>...")]
    ↓
UI_SPPostgameMenu_f():
  • Parse argc/argv → postgameMenuInfo struct
  • Fetch per-player names from configstrings
  • Compute tier/level/awards logic
  • Call UI_SetBestScore, UI_LogAwardData (persist to disk)
  • UI_PushMenu() activates menu
    ↓
[Each frame: Menu_Draw → UI_SPPostgameMenu_MenuDraw]
  Phase 1 (≥5s): Display podium names, play winner sound
    → Transition to Phase 2
  Phase 2 (numAwards × 2s + ≥5s hold): Draw medal animations with sound
    → Transition to Phase 3
  Phase 3: Activate buttons, optionally play tier cinematics, render scoreboard
    ↓
[User input: key or button click]
  UI_SPPostgameMenu_MenuKey() / button callbacks:
    • "Again" → map_restart 0
    • "Next" → compute next level (respecting tier bounds) → UI_SPArena_Start
    • "Menu" → disconnect; levelselect
```

## Learning Notes

**1. QVM Sandbox Architecture**
- This entire file communicates with the engine exclusively via indexed `trap_*` syscalls. The UI VM has **no direct access** to engine data or functions. This boundary is enforced by the VM interpreter (`code/qcommon/vm.c`), making the UI inherently sandboxed and crashproof.
- Modern engines often expose a simpler scripting API; Q3's numbered syscall table is more rigid but enables hot-reload and safety.

**2. Campaign Tier System**
- Q3's single-player progression is **tier-based**: 7 tiers, each with 4–6 arenas. Tier completion triggers cinematics and unlocks the next tier.
- `UI_SPPostgameMenu_NextEvent` implements the core tier logic: if the player hasn't unlocked the next tier, revert the "Next" button to the current tier's final arena. This prevents progression past unlocked content.

**3. Award System (Cosmetic Feedback)**
- Awards are **not computed here**; they're sent from the game VM as arguments. The UI's only job is to display them and persist via `UI_LogAwardData` (for career records).
- The FRAGS award is special: it's a cumulative milestone (every 100 frags), not per-game, so the value can grow across multiple sessions.

**4. Time-Driven UI without Blocking**
- Unlike frame-based games, Q3 UI has no guaranteed update tick. State progression is entirely **real-time–based** (`uis.realtime`). This is a key difference from modern engines that separate logic ticks from render frames.

**5. Scoreboard Scrolling (Elegant Constraint Handling)**
- Only 3 scoreboard rows are visible due to screen space. Rather than paging or scrolling, the file uses a rotating scroll (`timer / 1500 % numClients`) that cycles through all players. This is a simple, elegant solution to a UX problem.

**6. Cinematic Integration**
- Tier completion (`postgameMenuInfo.won == postgameMenuInfo.lastTier`) triggers a full-screen cinematic (`end.RoQ`) that fades to the main menu. Intermediate tier completions show tier-specific cinematics.
- The postgame menu serves as the **orchestration point** between gameplay and content flow.

## Potential Issues

**1. Hardcoded Array Bounds**
- `MAX_SCOREBOARD_CLIENTS = 8` is hardcoded. If a server sends more than 8 players' postgame data, the arrays `clientNums[]`, `ranks[]`, `scores[]` will overflow. The file caps at line `if (numClients > MAX_SCOREBOARD_CLIENTS)` but doesn't gracefully degrade.
- **Risk**: Low in practice (small servers), but fragile design. A bounds check with a warning would be safer.

**2. Name Truncation Heuristic**
- `Prepname()` truncates names until `UI_ProportionalStringWidth()` ≤ 256 pixels. If the font width changes or proportional rendering changes, names may not fit as expected.
- **Risk**: Minor (visual clipping), but string handling is a historical source of bugs.

**3. Server ID Race Condition**
- The file checks `serverId` against `sv_serverid` cvar each frame. However, if a server shuts down and a new server starts on the same machine in the same process, they could reuse the same ID briefly.
- **Risk**: Minimal (Q3 server IDs are 32-bit timestamps, very unlikely to collide within milliseconds), but theoretically possible.

**4. Global State Not Reset on Re-entry**
- If `UI_SPPostgameMenu_f` is called while the postgame menu is already active, `postgameMenuInfo` is overwritten. Phase state resets to 1, but other fields (e.g., `playedSound[]` array) are not cleared.
- **Risk**: If a sound is marked as played in round 1, and the postgame menu is called again, the same sound won't replay. Unlikely in practice but unsound initialization.

**5. Hard-Coded Asset Paths**
- All shader and sound paths are compile-time constants. If assets are renamed or moved, this file must be recompiled and relinked into the QVM.
- **Risk**: Not a runtime bug, but a maintenance burden. Modern content-driven UIs would parametrize these.
