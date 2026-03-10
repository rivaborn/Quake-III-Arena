# code/q3_ui/ui_confirm.c

## File Purpose
Implements a reusable modal confirmation dialog and message box for the Quake III Arena legacy UI (q3_ui). It presents a yes/no prompt or a multi-line informational message overlaid on the current screen, invoking a callback with the user's boolean result.

## Core Responsibilities
- Display a modal yes/no confirmation dialog with a question string
- Display a modal message box with multiple text lines and a single "OK" button
- Route keyboard input (`Y`/`N`, arrow keys, tab) to the appropriate menu items
- Pop the menu from the stack and invoke a caller-supplied callback with the result
- Cache the confirmation frame artwork via `trap_R_RegisterShaderNoMip`
- Support an optional custom draw callback for additional overlay rendering
- Determine fullscreen vs. overlay mode based on connection state

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `confirmMenu_t` | struct (typedef) | Holds all state for the active confirmation/message menu: menu framework, yes/no text items, slash position, question string, draw/action callbacks, style flags, and multi-line string array |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_confirm` | `confirmMenu_t` | static (file) | Singleton instance of the confirmation menu; reused for every confirm/message invocation |

## Key Functions / Methods

### ConfirmMenu_Event
- **Signature:** `static void ConfirmMenu_Event( void* ptr, int event )`
- **Purpose:** Callback fired when YES or NO is activated; pops the menu and invokes the action callback with the boolean result.
- **Inputs:** `ptr` — pointer to the activated `menucommon_s`; `event` — must be `QM_ACTIVATED` to proceed.
- **Outputs/Return:** void
- **Side effects:** Calls `UI_PopMenu()`, calls `s_confirm.action(result)` if set.
- **Calls:** `UI_PopMenu`, `s_confirm.action`
- **Notes:** Ignores all events except `QM_ACTIVATED`; result is `qfalse` for NO, `qtrue` for YES/OK.

### ConfirmMenu_Key
- **Signature:** `static sfxHandle_t ConfirmMenu_Key( int key )`
- **Purpose:** Custom key handler; remaps left/right arrows to Tab for navigation, and `Y`/`N` directly trigger YES/NO events.
- **Inputs:** `key` — raw key code.
- **Outputs/Return:** `sfxHandle_t` from `Menu_DefaultKey`.
- **Side effects:** May call `ConfirmMenu_Event` directly for `Y`/`N` keys.
- **Calls:** `ConfirmMenu_Event`, `Menu_DefaultKey`

### MessageMenu_Draw
- **Signature:** `static void MessageMenu_Draw( void )`
- **Purpose:** Draw function for the message box variant; renders the frame, iterates `s_confirm.lines`, then draws menu items and optional custom overlay.
- **Inputs:** None (reads `s_confirm` globals).
- **Outputs/Return:** void
- **Side effects:** Issues render calls via `UI_DrawNamedPic`, `UI_DrawProportionalString`, `Menu_Draw`, optional `s_confirm.draw()`.
- **Calls:** `UI_DrawNamedPic`, `UI_DrawProportionalString`, `Menu_Draw`, `s_confirm.draw`
- **Notes:** Function comment contains a typo ("MessaheMenu_Draw").

### ConfirmMenu_Draw
- **Signature:** `static void ConfirmMenu_Draw( void )`
- **Purpose:** Draw function for the yes/no confirmation variant; renders the frame, question string, a "/" separator at the computed `slashX` position, menu items, and optional overlay.
- **Inputs:** None (reads `s_confirm` globals).
- **Outputs/Return:** void
- **Side effects:** Render calls as above.
- **Calls:** `UI_DrawNamedPic`, `UI_DrawProportionalString`, `Menu_Draw`, `s_confirm.draw`

### UI_ConfirmMenu_Style
- **Signature:** `void UI_ConfirmMenu_Style( const char *question, int style, void (*draw)(void), void (*action)(qboolean result) )`
- **Purpose:** Primary initializer for the yes/no dialog; computes proportional string layout, builds menu items, and pushes the menu.
- **Inputs:** `question` — prompt string; `style` — text style flags; `draw` — optional extra draw callback; `action` — result callback.
- **Outputs/Return:** void
- **Side effects:** Zeroes `s_confirm`, registers art, pushes menu via `UI_PushMenu`, sets cursor to NO by default.
- **Calls:** `memset`, `ConfirmMenu_Cache`, `UI_ProportionalStringWidth`, `trap_GetClientState`, `Menu_AddItem`, `UI_PushMenu`, `Menu_SetCursorToItem`
- **Notes:** Default cursor is on NO (safe default for destructive confirmations).

### UI_ConfirmMenu
- **Signature:** `void UI_ConfirmMenu( const char *question, void (*draw)(void), void (*action)(qboolean result) )`
- **Purpose:** Convenience wrapper calling `UI_ConfirmMenu_Style` with `UI_CENTER|UI_INVERSE` style.
- **Calls:** `UI_ConfirmMenu_Style`

### UI_Message
- **Signature:** `void UI_Message( const char **lines )`
- **Purpose:** Initializes and pushes a message box (null-terminated string array) with a single OK button. Reuses `s_confirm` and the confirmation key handler.
- **Inputs:** `lines` — null-terminated array of strings to display.
- **Outputs/Return:** void
- **Side effects:** Zeroes `s_confirm`, pushes menu, sets cursor to OK/YES item.
- **Calls:** `memset`, `ConfirmMenu_Cache`, `UI_ProportionalStringWidth`, `trap_GetClientState`, `Menu_AddItem`, `UI_PushMenu`, `Menu_SetCursorToItem`
- **Notes:** Shares `ConfirmMenu_Key` for input, so pressing `Y` or Enter dismisses; `N` also triggers `ConfirmMenu_Event` with the YES id (since only one item is registered, `s_confirm.no` is zeroed/unused).

### ConfirmMenu_Cache
- **Signature:** `void ConfirmMenu_Cache( void )`
- **Purpose:** Pre-registers the `"menu/art/cut_frame"` shader asset.
- **Calls:** `trap_R_RegisterShaderNoMip`

## Control Flow Notes
These functions are called on-demand from other UI modules (e.g., disconnect confirm, exit confirm). `UI_ConfirmMenu` / `UI_ConfirmMenu_Style` / `UI_Message` push a new menu frame onto the UI stack. Per-frame rendering happens via the `menu.draw` callback (`ConfirmMenu_Draw` or `MessageMenu_Draw`). Input is routed through `menu.key` (`ConfirmMenu_Key`). On user selection, `ConfirmMenu_Event` pops the menu and returns control to the caller via the `action` callback.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`)
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `Menu_DefaultKey`, `Menu_Draw`, `UI_DrawNamedPic`, `UI_DrawProportionalString`, `UI_ProportionalStringWidth`, `trap_R_RegisterShaderNoMip`, `trap_GetClientState`, `color_red`, key constants (`K_TAB`, `K_LEFTARROW`, etc.)
