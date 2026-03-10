# code/q3_ui/ui_team.c — Enhanced Analysis

## Architectural Role

This file implements a modal in-game team selection dialog that bridges player intent (menu selection) to server-side game logic (team membership). It participates in three key subsystem interactions: (1) querying the **Server** subsystem's published game state (`CS_SERVERINFO`) to determine valid team modes, (2) dispatching commands back to the **Server** to change team affiliation, and (3) registering a display asset with the **Renderer** subsystem for async shader compilation. The menu is ephemerally created and destroyed on each invocation, embodying the stateless callback-driven pattern of the Q3 UI VM sandbox.

## Key Cross-References

### Incoming (who depends on this file)
- `code/q3_ui/ui_ingame.c` — likely calls `UI_TeamMainMenu()` when player selects "Team" from in-game menu
- Other `code/q3_ui/*.c` files that compose the main in-game UI surface

### Outgoing (what this file depends on)

**Syscalls to Engine (via `ui_local.h` → `ui_syscalls.c`)**:
- `trap_Cmd_ExecuteText()` — enqueues `cmd team {red|blue|free|spectator}` to command buffer; bridges to **qcommon/cmd.c** dispatcher
- `trap_GetConfigString(CS_SERVERINFO, ...)` — reads server-published game state; **Server** subsystem publishes this via `SV_SetConfigstring()` each frame
- `trap_R_RegisterShaderNoMip()` — registers shader asset `"menu/art/cut_frame"` with **Renderer** for pre-caching; prevents frame-one stalls

**UI Framework (from `code/q3_ui/*`)**:
- `UI_ForceMenuOff()` — defined in `ui_atoms.c`; pops all menus from stack immediately
- `UI_PushMenu()` — defined in `ui_atoms.c`; pushes menu onto active stack
- `Menu_AddItem()` — defined in `ui_qmenu.c`; registers widget into menu framework
- Global constants: `MTYPE_PTEXT`, `QMF_*` flags, `UI_CENTER`, `UI_SMALLFONT`, `colorRed` — all from `ui_local.h`

**External Utilities**:
- `Info_ValueForKey()` — defined in `code/game/q_shared.c`; parses configstring key-value pairs
- `atoi()` — C stdlib; converts game type enum string to integer

## Design Patterns & Rationale

### Callback-Driven Event Model
The menu implements a pure event-callback architecture: `TeamMain_MenuEvent()` is registered as the `callback` field of every text widget. The UI framework (qmenu) invokes this callback **only** when `QM_ACTIVATED` event fires (button press), allowing stateless, request-driven menu behavior with no polling loop.

### Gametype-Conditional State Graying
`TeamMain_MenuInit()` queries server game type once and adjusts `QMF_GRAYED` flags accordingly. This is a **server-time decision** (read once per menu open), not per-frame. The rationale: team options are invalid for certain gametypes (SP/FFA have no teams; CTF/Team have no free-play), so disabling them prevents nonsensical user selections. **Notably**, the spectate button is **never** grayed, implying it's always valid—a design choice reflecting that spectating transcends gametype rules.

### Ephemeral Menu Lifecycle
The menu is **created fresh** each time `UI_TeamMainMenu()` is called (via `TeamMain_MenuInit()` zeroing `s_teammain`). This is idiomatic to Q3's UI VM: no persistent state between menu invocations. Combined with `UI_ForceMenuOff()` on selection, the entire dialog is born and destroyed within a single user interaction—simplifying memory and input management.

### Shader Pre-Caching
`TeamMain_Cache()` is called during `TeamMain_MenuInit()` rather than lazily during first draw. This moves shader compilation to menu-open time instead of risking a hitch on first frame render, following Q3's pervasive pre-cache-on-init pattern seen throughout cgame and renderer.

## Data Flow Through This File

1. **Initialization**: `UI_TeamMainMenu()` called (from `ui_ingame.c` or equivalent) → `TeamMain_MenuInit()` called
2. **Server State Query**: `trap_GetConfigString(CS_SERVERINFO, ...)` retrieves configstring; `Info_ValueForKey()` extracts `"g_gametype"` value; `atoi()` converts to enum
3. **Conditional Graying**: Switch on gametype enum; set `QMF_GRAYED` flags on `joinred`/`joinblue` (if FFA/SP/Tournament) or `joingame` (if Team/CTF)
4. **Shader Registration**: `trap_R_RegisterShaderNoMip("menu/art/cut_frame")` queued for renderer
5. **Menu Population**: All five widgets (frame + four buttons) added to menu framework via `Menu_AddItem()` loop
6. **Push to Stack**: `UI_PushMenu(&s_teammain.menu)` makes menu active and visible
7. **User Input**: Player presses button → UI framework fires `QM_ACTIVATED` event → `TeamMain_MenuEvent()` invoked
8. **Command Dispatch**: `TeamMain_MenuEvent()` switches on button ID and calls `trap_Cmd_ExecuteText(EXEC_APPEND, "cmd team {red|blue|free|spectator}\n")`
9. **Menu Teardown**: `UI_ForceMenuOff()` called; menu popped from stack; `s_teammain` global remains in memory (static) but unused until next invocation

**Key state barrier**: Game type is read **once at init**, not each frame. This is safe because gametype is immutable during a connected session.

## Learning Notes

### Idiomatic Q3 UI VM Patterns
- **Syscall-only communication**: Zero direct function calls into engine; all bridging via `trap_*` opcode dispatch. This enforces sandbox isolation and allows the UI to be recompiled independently.
- **Stateless callback dispatch**: No polling loop or per-frame `think` function. Events are request-driven and synchronous.
- **Lazy initialization on display**: Menu is built fresh when shown, not prebuilt. Memory footprint is minimal since `s_teammain` is a static singleton created once and reused.

### Rule Synchronization Pattern
This file demonstrates a critical Q3 pattern: **the UI reads server rules to validate user intent**. The `CS_SERVERINFO` configstring is the **source of truth** for game mode. The UI doesn't hard-code rules; it queries them. This decouples UI from game logic and allows mods to change rules without recompiling UI.

### Comparison to Modern Engines
- **No data binding**: Modern engines use reactive frameworks (React, Vue, Unreal's UMG) with declarative state. This file is imperative: it reads once, sets flags, done.
- **No menu persistence**: Contemporary engines often keep menus loaded in memory for instant reshow. Q3 recreates menus on every open, trading memory for simplicity.
- **Explicit command dispatch**: No fancy RPC or event bus. A button press directly enqueues a text command into the engine's input buffer—low-level and transparent.

## Potential Issues

### Likely Bug: joinblue Color
Line 138: `s_teammain.joinblue.color = colorRed;` — should be `colorBlue`. All four button items initialize their color field, and the red button also uses `colorRed`, but the blue button mistakenly uses the same color. This is almost certainly copy-paste error. Result: visual mislabeling (blue button appears red).

### Missing Error Handling
`trap_GetConfigString()` returns an int (bytes read), not validated before passing buffer to `Info_ValueForKey()`. If the buffer is empty or malformed, `Info_ValueForKey()` may return `NULL` or garbage, and `atoi()` returns 0 (which is `GT_FFA`, a valid enum value but possibly not intended fallback). Defensive code would check return value and supply a sane default.

### No Spectate Graying in Any Gametype
The spectate option is never grayed, even for gametypes where spectatorship might not be semantically sensible (e.g., single-player). This is either by design (spectate is always valid) or an oversight. No other codebase context provided to disambiguate.

### Hard-Coded IDs in Global Namespace
`ID_JOINRED` (100), `ID_JOINBLUE` (101), etc. are file-static and isolated to this menu, so collision risk with other menus is low. However, if two menus are pushed simultaneously (unlikely in Q3 but theoretically possible), ID collisions could cause event misrouting. Modern systems use namespaced/scoped IDs.

### No Validation of Gametype Enum Bounds
`atoi()` converts any string to an integer. If `g_gametype` is corrupted or a mod adds new gametypes beyond the known `GT_*` constants, the switch statement has a `default` case that treats unknown types as `GT_TEAM` (teams enabled, free-play disabled). This is a reasonable fallback but not explicitly documented.
