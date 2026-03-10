# code/q3_ui/ui_rankings.c — Enhanced Analysis

## Architectural Role

This file implements the **GRank online rankings menu**, a critical UI integration point for Quake III Arena's optional global ranking system. It serves as the primary user-facing interface for ranking-aware servers, presenting context-sensitive options (login, logout, signup, spectate, leave) that reflect the player's current server-side ranking authentication state. The file is part of the legacy `code/q3_ui/` menu VM and communicates with the engine exclusively via indexed `trap_*` syscalls, never accessing engine internals directly. It bridges the server's `client_status` cvar (which tracks the player's GRank lifecycle state) to UI presentation logic.

## Key Cross-References

### Incoming (who depends on this file)
- **UI Framework (`ui_atoms.c`, `ui_qmenu.c`):** Calls `UI_RankingsMenu()` to activate the rankings popup; thereafter invokes `Rankings_MenuEvent()` as the menu event dispatcher when items are activated
- **Menu Widget System:** Calls the three custom owner-draw callbacks (`Rankings_DrawText`, `Rankings_DrawName`, `Rankings_DrawPassword`) as field-rendering overrides for text input widgets used by companion menus (login, signup)
- **Related UI Menus:** `UI_LoginMenu()`, `UI_SignupMenu()`, `UI_SetupMenu()` (defined elsewhere in `code/q3_ui/`) are pushed onto the stack by `Rankings_MenuEvent()` dispatcher
- **Menu Stack:** `UI_PushMenu()` and `UI_PopMenu()` called by the framework to activate/deactivate this menu

### Outgoing (what this file depends on)
- **Engine Trap Calls:**
  - `trap_Cvar_VariableValue("client_status")` — reads server-side GRank authentication state (NEW, SPECTATOR, VALIDATING, PENDING, LEAVING, etc.)
  - `trap_CL_UI_RankUserRequestLogout()` — initiates client-side logout handshake with GRank server
  - `trap_Cmd_ExecuteText(EXEC_APPEND, "...")` — executes console commands (`rank_spectate`, `disconnect`)
  - `trap_R_RegisterShaderNoMip()` — pre-caches frame background shader (`menu/art/cut_frame`)
  - `trap_Key_GetOverstrikeMode()` — queries keyboard overstrike mode for cursor rendering
- **UI Framework Functions:**
  - `UI_DrawChar()` — low-level 2D character rendering
  - `UI_LoginMenu()`, `UI_SignupMenu()`, `UI_SetupMenu()`, `UI_ForceMenuOff()` — navigation helpers
  - `Menu_AddItem()` — adds widgets to menu framework
  - `UI_PushMenu()` — pushes menu onto active stack
- **String/Memory Utilities:**
  - `Q_isalpha()`, `Q_CleanStr()`, `Q_strncpyz()`, `strlen()` — standard string operations
- **Global UI Constants:**
  - `g_color_table[ColorIndex(COLOR_WHITE)]`, `color_white`, `text_color_normal`, `text_color_highlight` — color palette shared across UI

## Design Patterns & Rationale

**State-Driven UI Visibility:**
The file reads `client_status` cvar on every menu init and conditionally shows/hides/grays items based on the player's GRank lifecycle. This is a classic pattern where server-authoritative state (transmitted via cvar) drives client-side UI presentation without tight coupling. The menu is never responsible for determining the player's actual ranking status—it simply queries the engine and reflects it.

**Menu Event Dispatcher:**
`Rankings_MenuEvent()` follows the callback-based dispatcher pattern: each menu item holds an `ID_*` constant, the callback checks event type (`QM_ACTIVATED`), and dispatches to a specific action (push new menu, execute command, call trap function). This decouples item definition from action handling.

**Owner-Draw Text Field Rendering:**
The three custom draw callbacks (`Rankings_DrawText`, `Rankings_DrawName`, `Rankings_DrawPassword`) override the default `MenuField_Draw` behavior. Rather than delegating to a generic draw function, they implement specialized rendering:
- `Rankings_DrawText`: Raw character-by-character rendering with blinking insert/overstrike cursor
- `Rankings_DrawName`: Sanitizes input (alphanumeric only, strips color codes), then draws via `Rankings_DrawText`
- `Rankings_DrawPassword`: Temporarily masks plaintext with `'*'` for secure display, then unmasks

This design allows the framework to support custom widgets without modifying the core framework code.

**Input Sanitization at UI Layer:**
Both `Rankings_DrawName` and `Rankings_DrawPassword` perform defensive character filtering in-place. Comments marked `GRANK_FIXME` suggest this may have been a temporary enforcement mechanism (pending a proper validation layer). The sanitization happens during rendering, which is slightly unusual—typically input validation would occur during key capture—but it ensures the buffer is always clean when read.

**Deferred Menu Initialization:**
`Rankings_MenuInit()` re-initializes the static `s_rankings` struct on every menu open, rather than initializing once at UI startup. This is safe and allows the menu to reflect current engine state (cvar values) without caching stale values.

## Data Flow Through This File

**Entry:**
1. Engine calls `UI_RankingsMenu()` (public entry point, likely triggered by user navigating in-game menu or auto-shown when joining a GRank-enabled server)
2. `UI_RankingsMenu()` → `Rankings_MenuInit()` → `Rankings_Cache()` (pre-register shader) + populate `s_rankings` struct
3. `UI_PushMenu(&s_rankings.menu)` places menu on the active stack

**Per-Frame:**
- Menu framework repeatedly calls custom draw callbacks if text fields have focus (though this menu has no text field input)
- User clicks a menu item → framework calls `Rankings_MenuEvent(ptr, QM_ACTIVATED)`

**Dispatch & Exit:**
- `Rankings_MenuEvent()` dispatches based on item ID:
  - `ID_LOGIN` → `UI_LoginMenu()` (pushes login dialog)
  - `ID_LOGOUT` → `trap_CL_UI_RankUserRequestLogout()` + `UI_ForceMenuOff()` (close menu, send logout)
  - `ID_CREATE` → `UI_SignupMenu()` (pushes signup dialog)
  - `ID_SPECTATE` → `trap_Cmd_ExecuteText("rank_spectate")` + `UI_ForceMenuOff()`
  - `ID_SETUP` → `UI_SetupMenu()` (hidden by FIXME comment; would push setup dialog)
  - `ID_LEAVE` → `trap_Cmd_ExecuteText("disconnect")` + `UI_ForceMenuOff()`

**State Transitions:**
The visibility of items depends on `client_status`:
- `NEW` or `SPECTATOR`: show LOGIN, CREATE, SPECTATE; hide LOGOUT
- Anything else (logged in): hide LOGIN/CREATE/SPECTATE; show LOGOUT
- `VALIDATING`, `PENDING`, or `LEAVING`: gray out LOGIN, CREATE, LOGOUT (disable interaction during transient states)

## Learning Notes

**Q3A UI Architecture:**
This file demonstrates how the Q3A UI VM framework works at runtime:
- Menus are passive data structures (`menuframework_s` + widget structs)
- The framework owns the event loop and rendering pipeline (in `ui_atoms.c`)
- Individual menus register callbacks (like `Rankings_MenuEvent`) rather than implementing their own loops
- Trap calls provide the only bridge to the engine; the UI VM never links against engine code

**Idiomatic Design of the Era:**
- **No async patterns:** Menu callbacks are synchronous; logout, spectate, and disconnect are fire-and-forget commands (actual state changes happen server-side)
- **Struct-of-arrays for widgets:** Each menu aggregates its widgets in a single struct (`rankings_t`), not a dynamic array. This is efficient and type-safe in C
- **Cvar-driven state:** The UI reads cvars to track server-side state, avoiding bidirectional coupling
- **Static globals with re-initialization:** The `s_rankings` static is cleared and re-populated on each menu open; no teardown phase is needed because the UI stack management handles cleanup

**Contrast with Modern Engines:**
- Modern engines typically use data-driven UI markup (HTML/XML) or a node-based scene graph; this uses hardcoded C struct initialization
- Modern engines often have async state queries; Q3A uses synchronous cvar reads
- Modern engines separate input validation from rendering; Q3A does both in the draw callback

**GRank System Architecture:**
This file reveals how GRank integrates into the runtime:
1. Server maintains `client_status` cvar reflecting the player's auth state
2. Client reads this cvar to determine which options are available
3. User-triggered actions (login, logout, spectate) are sent as commands or trap calls
4. The actual ranking state changes are server-authoritative and reported back via cvar updates

This is a classic server-authoritative architecture where the UI is purely a presentation layer.

## Potential Issues

**Input Sanitization Placement:**
`Rankings_DrawName` and `Rankings_DrawPassword` sanitize during rendering, not input capture. This means the buffer could be corrupted by rapid key presses before the next draw call. The `GRANK_FIXME` comments suggest the developers knew this was not ideal. A proper solution would sanitize in a key-press handler before the character reaches the buffer.

**Color Hardcoding in Rankings_DrawText:**
The function computes `color` based on focus state (normal vs. highlight), then immediately overwrites it with `g_color_table[ColorIndex(COLOR_WHITE)]`, ignoring the computed value. This appears to be dead code (the focus-based color is never used), suggesting incomplete refactoring.

**Setup Menu Hidden by Comment:**
The Setup item is unconditionally hidden with a `GRank FIXME -- don't need setup option any more` comment. However, the item is still initialized, added to the menu, and its event handler is defined. This suggests the code was left in place "just in case" rather than being removed.

**No Logout State Validation:**
When the user clicks LOGOUT, the code calls `trap_CL_UI_RankUserRequestLogout()` immediately without checking if the player is actually logged in. Relying on server-side validation is reasonable, but a client-side guard (`if (status == QGR_STATUS_LOGGED_IN)`) would provide faster feedback.
