# code/q3_ui/ui_mfield.c

## File Purpose
Implements low-level editable text field widgets for the Q3 UI menu system. Provides both a raw `mfield_t` editing core and a higher-level `menufield_s` wrapper that integrates with the `menuframework_s` item system.

## Core Responsibilities
- Render a scrollable, optionally blinking text field with cursor (`MField_Draw`)
- Handle keyboard navigation: left/right arrows, Home, End, Delete, Insert (overstrike toggle)
- Handle character input with insert/overstrike modes and optional maxchars limit
- Clipboard paste via `trap_GetClipboardData`
- Initialize `menufield_s` bounding box geometry for hit-testing and layout
- Draw a `menufield_s` with focus highlight, label, and cursor arrow glyph
- Route menu-system key events to the underlying `mfield_t` with case/digit filtering

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `mfield_t` | struct (typedef) | Raw editable field: buffer, cursor pos, scroll offset, width in chars, maxchars |
| `menufield_s` | struct (typedef) | Menu item wrapper combining `menucommon_s` generic header with an embedded `mfield_t` |

## Global / File-Static State
None.

## Key Functions / Methods

### MField_Draw
- **Signature:** `void MField_Draw( mfield_t *edit, int x, int y, int style, vec4_t color )`
- **Purpose:** Draws the visible portion of the field text and, if `UI_PULSE` is set in style, a blinking cursor glyph.
- **Inputs:** Field state, screen position in pixels, draw style flags, RGBA color.
- **Outputs/Return:** void
- **Side effects:** Mutates `edit->scroll` to guarantee cursor visibility. Calls `UI_DrawString`, `UI_DrawChar`.
- **Calls:** `trap_Error`, `memcpy`, `strlen`, `UI_DrawString`, `trap_Key_GetOverstrikeMode`, `UI_DrawChar`
- **Notes:** Cursor glyph 10 = insert bar, 11 = overstrike block. Adjusts x origin for `UI_CENTER`/`UI_RIGHT` alignment before placing the cursor.

### MField_Paste
- **Signature:** `void MField_Paste( mfield_t *edit )`
- **Purpose:** Reads OS clipboard (max 64 bytes) and feeds each character through `MField_CharEvent`, respecting insert/overstrike and maxchars.
- **Inputs:** Field to paste into.
- **Outputs/Return:** void
- **Side effects:** Calls `trap_GetClipboardData`, then `MField_CharEvent` per character.
- **Calls:** `trap_GetClipboardData`, `strlen`, `MField_CharEvent`

### MField_KeyDownEvent
- **Signature:** `void MField_KeyDownEvent( mfield_t *edit, int key )`
- **Purpose:** Handles non-printable key codes: delete, cursor movement, home/end, insert toggle, Shift+Insert paste.
- **Inputs:** Field, key code constant (`K_DEL`, `K_LEFTARROW`, etc.).
- **Outputs/Return:** void
- **Side effects:** Mutates `edit->cursor`, `edit->scroll`, `edit->buffer` (delete). Calls `trap_Key_SetOverstrikeMode`.
- **Calls:** `MField_Paste`, `strlen`, `memmove`, `trap_Key_IsDown`, `trap_Key_SetOverstrikeMode`, `tolower`
- **Notes:** Ctrl+A/E handled here as well as in `MField_CharEvent` (duplicated intent for key vs char paths).

### MField_CharEvent
- **Signature:** `void MField_CharEvent( mfield_t *edit, int ch )`
- **Purpose:** Handles printable characters and control-character shortcuts (Ctrl+V paste, Ctrl+C clear, Ctrl+H backspace, Ctrl+A home, Ctrl+E end).
- **Inputs:** Field, ASCII character code.
- **Outputs/Return:** void
- **Side effects:** Mutates `edit->buffer`, `edit->cursor`, `edit->scroll`. May call `MField_Paste` or `MField_Clear`.
- **Calls:** `MField_Paste`, `MField_Clear`, `strlen`, `memmove`, `trap_Key_GetOverstrikeMode`
- **Notes:** Characters below 32 (after control shortcuts handled) are silently dropped.

### MField_Clear
- **Signature:** `void MField_Clear( mfield_t *edit )`
- **Purpose:** Resets buffer to empty string, cursor and scroll to zero.
- **Side effects:** Writes to `edit->buffer[0]`, `edit->cursor`, `edit->scroll`.

### MenuField_Init
- **Signature:** `void MenuField_Init( menufield_s* m )`
- **Purpose:** Calls `MField_Clear` and computes `generic.left/top/right/bottom` bounding box based on font size and label width.
- **Calls:** `MField_Clear`, `strlen`

### MenuField_Draw
- **Signature:** `void MenuField_Draw( menufield_s *f )`
- **Purpose:** Renders a complete menu field item: focus highlight rect, left-arrow focus glyph, label string, and the inner `MField_Draw`.
- **Inputs:** Menu field item pointer.
- **Side effects:** Calls renderer via `UI_FillRect`, `UI_DrawChar`, `UI_DrawString`, `MField_Draw`. Reads `Menu_ItemAtCursor` to determine focus.
- **Calls:** `Menu_ItemAtCursor`, `UI_FillRect`, `UI_DrawChar`, `UI_DrawString`, `MField_Draw`

### MenuField_Key
- **Signature:** `sfxHandle_t MenuField_Key( menufield_s* m, int* key )`
- **Purpose:** Menu system key router. Remaps Enter→Tab to advance focus, passes char events through case/digit filters to `MField_CharEvent`, passes raw keycodes to `MField_KeyDownEvent`.
- **Inputs:** Menu field, pointer to key code (may be mutated to `K_TAB`).
- **Outputs/Return:** `menu_buzz_sound` if `QMF_NUMBERSONLY` rejects an alpha key; 0 otherwise.
- **Calls:** `MField_CharEvent`, `MField_KeyDownEvent`, `Q_islower`, `Q_isupper`, `Q_isalpha`

## Control Flow Notes
This file is purely event-driven. `MenuField_Draw` is called each frame by the menu framework's draw pass. `MenuField_Key` is called from the menu framework's key dispatch when this item has focus. Neither function participates in the game simulation loop.

## External Dependencies
- **`ui_local.h`** — pulls in `mfield_t`, `menufield_s`, `menucommon_s`, key constants, draw style flags, `MAX_EDIT_LINE`, `QMF_*` flags, color externs.
- **Defined elsewhere:** `trap_GetClipboardData`, `trap_Key_GetOverstrikeMode`, `trap_Key_SetOverstrikeMode`, `trap_Key_IsDown`, `trap_Error`, `UI_DrawString`, `UI_DrawChar`, `UI_FillRect`, `Menu_ItemAtCursor`, `Q_islower`, `Q_isupper`, `Q_isalpha`, `menu_buzz_sound`, color arrays (`text_color_disabled`, `text_color_normal`, `text_color_highlight`, `listbar_color`).
