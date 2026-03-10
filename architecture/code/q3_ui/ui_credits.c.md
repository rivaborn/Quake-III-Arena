# code/q3_ui/ui_credits.c

## File Purpose
Implements the credits screen menu for Quake III Arena's legacy UI (`q3_ui`). It renders a static list of id Software team members and pushes itself onto the menu stack as a fullscreen menu that quits the game on any keypress.

## Core Responsibilities
- Define and register the credits menu structure with the UI menu system
- Draw all credit text (roles and names) using proportional string rendering
- Handle key input by triggering a game quit command
- Push the credits screen onto the active menu stack as fullscreen

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `creditsmenu_t` | struct | Thin wrapper around `menuframework_s`; holds the credits menu state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_credits` | `creditsmenu_t` | static (file) | Singleton credits menu instance; zeroed on each call to `UI_CreditMenu` |

## Key Functions / Methods

### UI_CreditMenu_Key
- **Signature:** `static sfxHandle_t UI_CreditMenu_Key( int key )`
- **Purpose:** Key handler for the credits menu; any non-character key causes the game to quit.
- **Inputs:** `key` — raw key code (may include `K_CHAR_FLAG`)
- **Outputs/Return:** `sfxHandle_t` — always returns `0` (no sound played)
- **Side effects:** Appends `"quit\n"` to the command buffer via `trap_Cmd_ExecuteText(EXEC_APPEND, ...)`
- **Calls:** `trap_Cmd_ExecuteText`
- **Notes:** Character events (`K_CHAR_FLAG` set) are silently ignored to avoid double-firing on key+char sequences.

### UI_CreditMenu_Draw
- **Signature:** `static void UI_CreditMenu_Draw( void )`
- **Purpose:** Renders all credit lines directly to screen using proportional and small character string draw calls. No scrolling; layout is computed by manually advancing a `y` coordinate.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Issues renderer draw calls via `UI_DrawProportionalString` and `UI_DrawString`
- **Calls:** `UI_DrawProportionalString`, `UI_DrawString`
- **Notes:** Section headers use a `1.42×` line-height multiplier for spacing; the trailing legal/contact line uses `UI_DrawString` with `color_red` instead of the proportional variant. All text is centered at x=320 (half of the 640-wide virtual screen).

### UI_CreditMenu
- **Signature:** `void UI_CreditMenu( void )`
- **Purpose:** Entry point. Initializes the credits menu and pushes it onto the UI menu stack.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Zeroes `s_credits`, assigns `draw`/`key` callbacks, sets `fullscreen = qtrue`, and calls `UI_PushMenu`
- **Calls:** `memset`, `UI_PushMenu`
- **Notes:** Declared `extern` in `ui_local.h`; called by the main menu or post-game flow to trigger credits display.

## Control Flow Notes
`UI_CreditMenu` is called during the UI **init/navigation** phase (not per-frame directly). Once pushed, the menu system calls `UI_CreditMenu_Draw` every frame to render, and routes input events to `UI_CreditMenu_Key`. There is no update/tick logic. On any non-character keypress the game exits immediately via the quit command.

## External Dependencies
- **Includes:** `ui_local.h` → transitively pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`
- **Defined elsewhere:**
  - `UI_DrawProportionalString`, `UI_DrawString` — `ui_atoms.c`
  - `UI_PushMenu` — `ui_atoms.c`
  - `trap_Cmd_ExecuteText` — `ui_syscalls.c`
  - `color_white`, `color_red` — `ui_qmenu.c`
  - `menuframework_s`, `K_CHAR_FLAG`, `PROP_HEIGHT`, `PROP_SMALL_SIZE_SCALE`, `SMALLCHAR_HEIGHT` — `ui_local.h` / `q_shared.h`
