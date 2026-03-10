# code/q3_ui/ui_spreset.c

## File Purpose
Implements the single-player "Reset Game" confirmation dialog for Quake III Arena's UI module. It presents a YES/NO prompt to the player and, on confirmation, wipes all single-player progress data and restarts the level menu from the beginning.

## Core Responsibilities
- Renders the reset confirmation dialog with a decorative frame and warning text
- Handles YES/NO menu item selection via mouse and keyboard (including `Y`/`N` hotkeys)
- On confirmation: calls `UI_NewGame()`, resets `ui_spSelection` to 0, pops the current menu stack entries, and re-launches the SP level menu
- Caches the background frame shader on demand
- Positions the `YES / NO` text layout dynamically using proportional string width calculations
- Sets fullscreen vs. overlay mode based on whether a game session is currently connected

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `resetMenu_t` | struct | Aggregates the menu framework, YES/NO text items, and the pre-computed slash separator X position |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_reset` | `resetMenu_t` | static (file) | Sole instance of the reset menu; zero-initialised on each `UI_ResetMenu()` call |

## Key Functions / Methods

### Reset_MenuEvent
- **Signature:** `void Reset_MenuEvent(void* ptr, int event)`
- **Purpose:** Callback fired when YES or NO is activated; performs the actual game reset or cancels.
- **Inputs:** `ptr` — pointer to the triggering `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** void
- **Side effects:** Pops one or two menu layers; if YES: calls `UI_NewGame()`, sets cvar `ui_spSelection` to 0, pops another menu layer, opens `UI_SPLevelMenu()`
- **Calls:** `UI_PopMenu`, `UI_NewGame`, `trap_Cvar_SetValue`, `UI_SPLevelMenu`
- **Notes:** Silently returns on any event other than `QM_ACTIVATED`; the NO path only pops once, the YES path pops twice (reset menu + caller menu).

### Reset_MenuKey
- **Signature:** `static sfxHandle_t Reset_MenuKey(int key)`
- **Purpose:** Custom key handler adding `Y`/`N` hotkeys and remapping arrow keys to `K_TAB` for lateral navigation.
- **Inputs:** `key` — raw key code
- **Outputs/Return:** `sfxHandle_t` from `Menu_DefaultKey`
- **Side effects:** May trigger `Reset_MenuEvent` directly for `Y`/`N` keys
- **Calls:** `Reset_MenuEvent`, `Menu_DefaultKey`
- **Notes:** Arrow keys (`K_LEFTARROW`, `K_RIGHTARROW`, numpad equivalents) are aliased to `K_TAB` before falling through to default handling.

### Reset_MenuDraw
- **Signature:** `static void Reset_MenuDraw(void)`
- **Purpose:** Draws the full reset dialog: decorative frame, "RESET GAME?" header, slash separator, YES/NO items, and four-line warning text block.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Issues multiple renderer draw calls
- **Calls:** `UI_DrawNamedPic`, `UI_DrawProportionalString`, `Menu_Draw`
- **Notes:** Slash position (`s_reset.slashX`) is baked at init time; warning text uses `PROP_HEIGHT` row spacing.

### Reset_Cache
- **Signature:** `void Reset_Cache(void)`
- **Purpose:** Pre-registers the decorative frame shader with the renderer.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Registers `"menu/art/cut_frame"` via `trap_R_RegisterShaderNoMip`
- **Calls:** `trap_R_RegisterShaderNoMip`

### UI_ResetMenu
- **Signature:** `void UI_ResetMenu(void)`
- **Purpose:** Entry point; initialises and pushes the reset confirmation menu onto the UI menu stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Zeroes `s_reset`, calls `Reset_Cache`, queries client connection state, configures both menu items, pushes menu, sets default cursor to NO
- **Calls:** `memset`, `Reset_Cache`, `UI_ProportionalStringWidth`, `trap_GetClientState`, `Menu_AddItem`, `UI_PushMenu`, `Menu_SetCursorToItem`
- **Notes:** Default cursor is placed on NO as a safety measure against accidental confirmation. Fullscreen mode is toggled based on `cstate.connState >= CA_CONNECTED`.

## Control Flow Notes
Activated by the SP level menu when the player chooses to reset progress. Runs entirely within the UI frame loop: `UI_ResetMenu` → `UI_PushMenu` registers `Reset_MenuDraw` and `Reset_MenuKey` as the active menu's callbacks, which are then invoked each frame by `UI_Refresh` → `Menu_Draw` / key dispatch. On completion, `Reset_MenuEvent` unwinds the stack and hands control to `UI_SPLevelMenu`.

## External Dependencies
- `ui_local.h` — pulls in all UI framework types, menu item types, trap syscalls, draw utilities, and SP game info functions
- **Defined elsewhere:** `UI_NewGame` (`ui_gameinfo.c`), `UI_SPLevelMenu` (`ui_spLevel.c`), `UI_PopMenu` / `UI_PushMenu` / `UI_DrawNamedPic` / `UI_DrawProportionalString` / `UI_ProportionalStringWidth` (`ui_atoms.c`), `Menu_Draw` / `Menu_DefaultKey` / `Menu_AddItem` / `Menu_SetCursorToItem` (`ui_qmenu.c`), `trap_*` syscall wrappers (`ui_syscalls.c`), `trap_R_RegisterShaderNoMip` (renderer via VM syscall)
