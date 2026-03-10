# code/q3_ui/ui_teamorders.c — Enhanced Analysis

## Architectural Role

This file implements an in-game team communication menu within the **UI VM** subsystem (`code/q3_ui`). It sits at the intersection of three domains: (1) the modal UI menu framework, (2) server state queries (player enumeration, gametype detection), and (3) the command execution layer for dispatching player intent back to the server. Unlike main-menu screens, this menu is triggered during active gameplay and converts a two-step user selection into a `say_team` console command—effectively using the chat infrastructure as the protocol for bot orders.

## Key Cross-References

### Incoming (who depends on this file)

- **ui_main.c** — calls `UI_TeamOrdersMenu()` to activate the menu and `UI_TeamOrdersMenu_f()` as a console command handler during gameplay (e.g., bound to a key like `+use_teamorders`)
- **Menu framework** — `UI_TeamOrdersMenu_Key` is registered as the custom key dispatcher (overriding default list handling) and `UI_TeamOrdersMenu_ListEvent` is the activation callback

### Outgoing (what this file depends on)

- **qcommon subsystem:**
  - `trap_GetConfigString(CS_SERVERINFO, ...)` — reads server info (gametype, max clients)
  - `trap_GetConfigString(CS_PLAYERS + n, ...)` — reads all connected player config strings to enumerate bots
  - `trap_GetClientState(...)` — reads local player client number and determines player's team
  - `trap_Cmd_ExecuteText(EXEC_APPEND, ...)` — executes the `say_team` command on local client
  - `Info_ValueForKey()` — parses name/team/skill fields from config strings
  - `Com_sprintf()` / `va()` — format `say_team` messages with bot name interpolation

- **Renderer (trap_R_RegisterShaderNoMip)** — pre-caches artwork shaders during menu init

- **UI framework (ui_qmenu.c / ui_atoms.c):**
  - `Menu_AddItem()`, `Menu_ItemAtCursor()`, `Menu_DefaultKey()` — standard menu widget lifecycle
  - `UI_DrawProportionalString()` — owner-draw rendering with color/style control
  - `UI_CursorInRect()` — hit-testing for mouse clicks
  - `UI_PushMenu()` / `UI_PopMenu()` — modal menu stack management

- **q_shared.c:** `Q_strncpyz()`, `Q_CleanStr()` (string sanitization), `atoi()` (integer parsing)

## Design Patterns & Rationale

### 1. **Trap Sandbox Boundary**
Every line of external communication flows through indexed `trap_*` syscalls. This reflects the VM sandbox architecture: the UI VM is untrusted bytecode and must ask the engine for any capability (state, rendering, command execution). This is idiomatic to late-1990s/early-2000s game engines and remains structurally sound for capability-based security.

### 2. **Two-Phase State Machine via List Switching**
Rather than maintaining explicit state variables (`STATE_SELECT_BOT`, `STATE_SELECT_ORDER`), the code reuses a single `menulist_s` widget and mutates its `itemnames`/`numitems` via `UI_TeamOrdersMenu_SetList()`. The phase is implicit: if the list shows bot names, clicking advances to orders; if it shows orders, clicking dispatches the message. This is memory-efficient (single widget allocation, no dynamic creation) and leverages the existing menu framework.

### 3. **Owner-Draw for Custom Rendering**
The list drawing is delegated to `UI_TeamOrdersMenu_ListDraw()`, which bypasses the framework's default list renderer and applies custom styling (yellow highlight with pulse, orange text for non-selected). This callback pattern is standard in retained-mode UI frameworks, adapted here to Quake's immediate-mode heritage.

### 4. **Command Transport via Chat**
Bot orders are delivered via `say_team "<message>"` rather than a custom game protocol. This reuses the existing chat message routing: local client sends a console command → server relays it as a chat message → cgame (and AI on server) receives it as player input. Clever layering that avoids new RPC definitions.

### 5. **Config String Polling**
Instead of registering a callback on server state changes, the menu reads snapshots of `CS_SERVERINFO` and `CS_PLAYERS` on every activation. This is simpler than event subscription and acceptable for a menu that is rarely opened, but reflects era constraints (QVMs have limited ability to register callbacks).

## Data Flow Through This File

**Phase 1: Menu Activation**
1. Player invokes via console command (`teamorders`) → `UI_TeamOrdersMenu_f()`
2. Guard checks: validate gametype ≥ `GT_TEAM`, player is not spectator
3. Call `UI_TeamOrdersMenu_Init()`
   - Zero `teamOrdersMenuInfo` state
   - Call `UI_TeamOrdersMenu_BuildBotList()`
     - Read `CS_SERVERINFO` for gametype and max clients
     - Read `CS_PLAYERS[i]` for each slot; filter: skip self, skip non-bots (skill == 0), skip other teams
     - Populate `botNames[]` array with cleaned names
     - Store enumerated `numBots` and `gametype`
   - Initialize all widgets (banner, frame, back button, list)
   - Initialize list to bot mode: `UI_TeamOrdersMenu_SetList(ID_LIST_BOTS)`
   - Push menu onto stack

**Phase 2: Bot Selection**
- User clicks or presses arrow key on list
- `UI_TeamOrdersMenu_Key()` handles navigation or click hit-testing
- On activation (Enter or click), `UI_TeamOrdersMenu_ListEvent()` is called
  - Detect `ID_LIST_BOTS` activation
  - Store selected bot index in `teamOrdersMenuInfo.selectedBot`
  - Transition list: `UI_TeamOrdersMenu_SetList(ID_LIST_CTF_ORDERS or ID_LIST_TEAM_ORDERS)` based on gametype
  - Return (menu stays open)

**Phase 3: Order Selection & Dispatch**
- User selects an order from the now-displayed list
- `UI_TeamOrdersMenu_ListEvent()` called again
  - Detect `ID_LIST_CTF_ORDERS` or `ID_LIST_TEAM_ORDERS` activation
  - Format message: `Com_sprintf(message, sizeof(message), ctfMessages[selection], botNames[selectedBot])`
    - e.g., `"Defend the Base" → "%s defend the base" → "Mynx defend the base"`
  - Execute: `trap_Cmd_ExecuteText(EXEC_APPEND, "say_team \"Mynx defend the base\"\n")`
  - Pop menu

**Rendering (Per-Frame)**
- `UI_TeamOrdersMenu_ListDraw()` is called by menu framework each frame
- Draw `itemnames[i]` for each list item at hardcoded `x = 320` (screen center)
- Apply color: yellow + pulse if selected and focused, orange otherwise
- Advance `y` by `PROP_HEIGHT` per item

## Learning Notes

### What Would a Developer Learn?

1. **Menu stacks are simple but powerful**: The push/pop discipline keeps context isolated. The custom key handler is the escape valve when framework behavior doesn't fit.

2. **Config strings are the state transport**: Rather than RPC calls, the engine pushes all dynamic state (player list, game rules) into named config strings readable by all VMs. Very effective for decoupling.

3. **Command-based UI is still practical**: "Just execute a console command" sidesteps the need for a type-safe game protocol for UI actions. Works as long as the receiving side (game VM) parses the command safely.

4. **Widget reuse via data binding**: The single list widget can present different data by swapping `itemnames`. This is close to modern data-binding patterns, though without reactive updates.

### Era-Specific Idioms

- **Hardcoded pixel coordinates** (e.g., `x = 320`): No layout engine; everything is manually positioned for 640×480. Modern UI systems use flexbox or layout trees.
- **Singleton global state** (`teamOrdersMenuInfo`): No dependency injection or context objects. Simple but inflexible.
- **Callback-driven rendering**: No scene graph or retained-mode tree; all drawing is immediate-mode callbacks. Reflects the era's renderer design (Quake III's back-end command queue).
- **Array-fixed limits** (9 bots max, 16-char names): No dynamic allocation in VMs (overhead, fragmentation). Hard limits reflect memory budgets.

## Potential Issues

1. **Team filtering logic is confusing** (lines 348–361):
   ```c
   playerTeam = TEAM_SPECTATOR;  // default init
   if( n == cs.clientNum ) {
       playerTeam = *Info_ValueForKey( info, "t" );
       continue;  // exits immediately
   }
   // botTeam check uses stale playerTeam = TEAM_SPECTATOR
   ```
   The intent appears to be: read the local player's team once, then filter bots to that team. However, `playerTeam` is always `TEAM_SPECTATOR` when checking `botTeam != playerTeam`. This likely works (filtering out bots on opposite teams, though the logic is opaque) but should be refactored for clarity—probably intended to read team once before the loop.

2. **No bounds checking on `numBots`**: If a server has 10+ human players and many bots, only the first 8 bots are shown (array size is 9 including "Everyone"). Silent truncation.

3. **Array initialization relies on stable positions**: `botNames[0]` is hardcoded to "Everyone"; if `numBots` ever exceeds 9, earlier entries get overwritten. No safety margin.

4. **Missing null-termination assertions**: `botNames[n]` is truncated to 16 bytes; if a bot name is exactly 16+ bytes, it's silently cut. No warning to the user or logging.

5. **Integer underflow in key navigation**: Line 187 wraps index to `numitems - 1` without checking if `numitems > 0`. Safe in practice (only reachable if list is non-empty) but fragile.

---

## Summary

This file is a textbook example of a **trap-based UI module within a QVM sandbox**—design idioms common to Quake III and many games of that era. It efficiently reuses framework components (single list widget with data swapping), leverages existing infrastructure (config strings for state, commands for control), and demonstrates that capability-based sandboxing (trap boundaries) remains a viable architecture. The two-phase selection pattern is elegant state-machine-via-data-switching, and the command-transport approach cleanly separates UI from game logic.
