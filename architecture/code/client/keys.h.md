# code/client/keys.h

## File Purpose
Declares the key input subsystem interface for the Quake III Arena client, defining key state storage, text input field operations, and the public API for key binding management.

## Core Responsibilities
- Defines the `qkey_t` struct representing per-key state (down/repeat/binding)
- Declares the global `keys[MAX_KEYS]` array as the central key state table
- Exposes text input field rendering and event functions for console/chat UI
- Declares the command history ring buffer and active console/chat fields
- Provides the public API for reading, writing, and querying key bindings
- Exposes insert/overstrike mode toggle state

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `qkey_t` | struct | Per-key state: whether pressed, autorepeat count, and bound command string |
| `keyNum_t` | enum (in keycodes.h) | Canonical integer codes for all keys including keyboard, mouse, joystick, and aux inputs |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `keys[MAX_KEYS]` | `qkey_t[256]` | global (extern) | Master key state table indexed by `keyNum_t` |
| `key_overstrikeMode` | `qboolean` | global (extern) | Tracks insert vs. overstrike mode for text fields |
| `historyEditLines[COMMAND_HISTORY]` | `field_t[32]` | global (extern) | Ring buffer of previously entered console commands |
| `g_consoleField` | `field_t` | global (extern) | Active console text input field |
| `chatField` | `field_t` | global (extern) | Active in-game chat text input field |
| `anykeydown` | `qboolean` | global (extern) | True if any key is currently held down |
| `chat_team` | `qboolean` | global (extern) | True if current chat is team-only |
| `chat_playerNum` | `int` | global (extern) | Player number associated with current chat |

## Key Functions / Methods

### Field_KeyDownEvent
- Signature: `void Field_KeyDownEvent(field_t *edit, int key)`
- Purpose: Handles a key-down event directed at a text input field (console or chat).
- Inputs: Pointer to the target `field_t`; raw key code.
- Outputs/Return: void
- Side effects: Mutates `field_t` buffer and cursor state.
- Calls: Defined elsewhere (`cl_keys.c`).
- Notes: Handles editing keys (backspace, arrows, delete, home, end, history navigation).

### Field_CharEvent
- Signature: `void Field_CharEvent(field_t *edit, int ch)`
- Purpose: Inserts a printable character into a text field at the current cursor position.
- Inputs: Pointer to the target `field_t`; ASCII character code.
- Outputs/Return: void
- Side effects: Mutates `field_t` buffer and advances cursor.
- Calls: Defined elsewhere.
- Notes: Respects overstrike mode.

### Field_Draw / Field_BigDraw
- Signature: `void Field_Draw(field_t *edit, int x, int y, int width, qboolean showCursor)` / `void Field_BigDraw(...)`
- Purpose: Renders a text input field to the screen at the given position; `BigDraw` uses a larger font.
- Inputs: Field pointer, screen coordinates, display width, cursor visibility flag.
- Outputs/Return: void
- Side effects: Issues renderer draw calls.
- Calls: Defined elsewhere (renderer/UI layer).

### Key_WriteBindings
- Signature: `void Key_WriteBindings(fileHandle_t f)`
- Purpose: Serializes all current key bindings to an open file (typically `q3config.cfg`).
- Inputs: Open file handle.
- Outputs/Return: void
- Side effects: File I/O.
- Calls: Defined elsewhere (`cl_keys.c`).

### Key_SetBinding / Key_GetBinding
- Signature: `void Key_SetBinding(int keynum, const char *binding)` / `char *Key_GetBinding(int keynum)`
- Purpose: Assigns or retrieves the console command string bound to a key index.
- Inputs: Key index (0–255); optional binding string.
- Outputs/Return: `Key_GetBinding` returns pointer into `keys[keynum].binding`; may be NULL.
- Side effects: `Key_SetBinding` frees old binding string and allocates new one.
- Calls: Defined elsewhere.

### Key_GetKey
- Signature: `int Key_GetKey(const char *binding)`
- Purpose: Reverse-lookup: finds the first key number bound to a given command string.
- Inputs: Binding string to search for.
- Outputs/Return: Key number, or -1 if not found.
- Side effects: None.
- Calls: Defined elsewhere.

### Key_IsDown / Key_ClearStates
- **`Key_IsDown`**: Returns whether the specified key is currently pressed.
- **`Key_ClearStates`**: Resets all key down/repeat states (called on focus loss or map load).

## Control Flow Notes
This header is consumed by `cl_keys.c` (implementation) and any client subsystem needing key state or binding queries (input processing, UI, console). It sits in the client's per-frame input path: raw OS events → `Key_Event()` → updates `keys[]` → `Field_KeyDownEvent`/`Field_CharEvent` for text modes → game/UI command dispatch.

## External Dependencies
- `../ui/keycodes.h` — defines `keyNum_t` enum covering all 256 possible key slots
- `field_t` — declared in `qcommon/qcommon.h` (noted inline by TTimo)
- `fileHandle_t` — defined in `q_shared.h` / `qcommon.h`
- `qboolean` — defined in `q_shared.h`
