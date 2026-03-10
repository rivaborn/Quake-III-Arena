# code/client/cl_keys.c

## File Purpose
Implements the client-side keyboard input system for Quake III Arena, managing key bindings, key state tracking, text field editing (console/chat), and dispatching input events to the appropriate subsystem (console, UI VM, cgame VM, or game commands).

## Core Responsibilities
- Maintain the `keys[]` array of key states (down, repeats, binding)
- Translate between key name strings and key numbers (bidirectionally)
- Handle console field and chat field line editing (cursor, scrolling, history)
- Dispatch key-down/key-up events to the correct handler based on `cls.keyCatchers`
- Execute bound commands (immediate and `+button` style with up/down pairing)
- Register `bind`, `unbind`, `unbindall`, `bindlist` console commands
- Write key bindings to config files

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `keyname_t` | struct | Maps a string name (e.g. `"TAB"`) to a keynum integer |
| `field_t` | struct (defined in `keys.h`) | Editable text field with cursor, scroll, buffer, widthInChars |
| `qkey_t` | struct (defined in `keys.h`) | Per-key state: `down`, `repeats`, `binding` string pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `keys` | `qkey_t[MAX_KEYS]` | global | All key states and bindings |
| `keynames` | `keyname_t[]` | file-static (array) | Name↔keynum lookup table |
| `g_consoleField` | `field_t` | global | Current console input line |
| `chatField` | `field_t` | global | In-game chat input line |
| `historyEditLines` | `field_t[COMMAND_HISTORY]` | global | Circular command history buffer |
| `nextHistoryLine` | `int` | global | Next write index into history ring |
| `historyLine` | `int` | global | Currently displayed history index |
| `key_overstrikeMode` | `qboolean` | global | Insert vs. overstrike toggle |
| `anykeydown` | `qboolean` | global | Count of keys currently held down |
| `chat_team` | `qboolean` | global | Whether chat is team-only |
| `chat_playerNum` | `int` | global | Target player for `tell` command (-1 = broadcast) |

## Key Functions / Methods

### Field_VariableSizeDraw
- **Signature:** `void Field_VariableSizeDraw(field_t *edit, int x, int y, int width, int size, qboolean showCursor)`
- **Purpose:** Renders an edit field with horizontal scrolling and a blinking cursor.
- **Inputs:** Field pointer, screen position, character size (`SMALLCHAR_WIDTH` or `BIGCHAR_WIDTH`), cursor visibility flag.
- **Outputs/Return:** None.
- **Side effects:** Calls `SCR_DrawSmallStringExt` / `SCR_DrawBigString` / `SCR_DrawSmallChar`; reads `cls.realtime` for blink timing; may modify `edit->scroll`.
- **Calls:** `Com_Error`, `Com_Memcpy`, `SCR_DrawSmallStringExt`, `SCR_DrawBigString`, `SCR_DrawSmallChar`, `Q_PrintStrlen`.
- **Notes:** `Field_Draw` and `Field_BigDraw` are thin wrappers forwarding to this function.

### Field_KeyDownEvent
- **Signature:** `void Field_KeyDownEvent(field_t *edit, int key)`
- **Purpose:** Handles non-printable key presses (arrows, home, end, delete, insert, shift-insert paste) for any edit field.
- **Inputs:** Edit field pointer, keynum.
- **Outputs/Return:** None.
- **Side effects:** Mutates `edit->cursor`, `edit->scroll`, `edit->buffer`; toggles `key_overstrikeMode` on K_INS; calls `Field_Paste`.
- **Calls:** `Field_Paste`, `memmove`.

### Field_CharEvent
- **Signature:** `void Field_CharEvent(field_t *edit, int ch)`
- **Purpose:** Inserts a printable character into an edit field, handling ctrl shortcuts (ctrl-v paste, ctrl-c clear, ctrl-h backspace, ctrl-a home, ctrl-e end).
- **Inputs:** Edit field pointer, ASCII character value.
- **Outputs/Return:** None.
- **Side effects:** Mutates `edit->buffer`, `edit->cursor`, `edit->scroll`; calls `Field_Paste` / `Field_Clear`.
- **Calls:** `Field_Paste`, `Field_Clear`, `memmove`.

### Console_Key
- **Signature:** `void Console_Key(int key)`
- **Purpose:** Handles key events when the console is active: executes commands, manages history navigation, scrolls the console buffer.
- **Inputs:** Keynum.
- **Outputs/Return:** None.
- **Side effects:** Calls `Cbuf_AddText` to enqueue commands; updates history ring; calls `Con_PageUp/Down/Top/Bottom`; calls `SCR_UpdateScreen` when disconnected.
- **Calls:** `Cbuf_AddText`, `Field_CompleteCommand`, `Field_Clear`, `Con_PageUp`, `Con_PageDown`, `Con_Top`, `Con_Bottom`, `SCR_UpdateScreen`, `Field_KeyDownEvent`, `Com_Printf`, `Com_sprintf`, `Q_strncpyz`.
- **Notes:** Non-slash input is treated as `cmd say` chat when in-game.

### Message_Key
- **Signature:** `void Message_Key(int key)`
- **Purpose:** Handles key events for the in-game chat/message input mode.
- **Inputs:** Keynum.
- **Outputs/Return:** None.
- **Side effects:** Calls `CL_AddReliableCommand` with `say`, `say_team`, or `tell` commands; clears `KEYCATCH_MESSAGE` on send or ESC.
- **Calls:** `Field_Clear`, `Com_sprintf`, `CL_AddReliableCommand`, `Field_KeyDownEvent`.

### Key_StringToKeynum
- **Signature:** `int Key_StringToKeynum(char *str)`
- **Purpose:** Converts a key name string (named key, single char, or `0xNN` hex) to an integer keynum.
- **Inputs:** Key name string.
- **Outputs/Return:** Integer keynum, or `-1` if not found.
- **Calls:** `Q_stricmp`.

### Key_KeynumToString
- **Signature:** `char *Key_KeynumToString(int keynum)`
- **Purpose:** Converts a keynum to its canonical name string.
- **Inputs:** Integer keynum.
- **Outputs/Return:** Static string (reused across calls — not thread-safe).
- **Notes:** Returns `"<KEY NOT FOUND>"` or `"<OUT OF RANGE>"` for invalid inputs.

### Key_SetBinding / Key_GetBinding
- **Signature:** `void Key_SetBinding(int keynum, const char *binding)` / `char *Key_GetBinding(int keynum)`
- **Purpose:** Set or retrieve the command string bound to a key; setting marks `cvar_modifiedFlags |= CVAR_ARCHIVE` to trigger config save.
- **Side effects:** `Z_Free` old binding, `CopyString` new one.

### CL_KeyEvent
- **Signature:** `void CL_KeyEvent(int key, qboolean down, unsigned time)`
- **Purpose:** Primary input dispatcher called by the OS/platform layer for every key press and release.
- **Inputs:** Keynum, up/down state, timestamp.
- **Outputs/Return:** None.
- **Side effects:** Updates `keys[key].down`, `keys[key].repeats`, `anykeydown`; dispatches to `Console_Key`, `Message_Key`, `VM_Call(uivm/cgvm)`, or `Cbuf_AddText` for bound actions; handles ESC specially; handles Linux Alt+Enter for fullscreen toggle.
- **Calls:** `CL_AddKeyUpCommands`, `VM_Call`, `Console_Key`, `Message_Key`, `Con_ToggleConsole_f`, `Key_ClearStates`, `Cvar_VariableValue`, `Cvar_Set`, `Cbuf_ExecuteText`, `CL_Disconnect_f`, `S_StopAllSounds`, `Com_sprintf`, `Cbuf_AddText`, `Com_Printf`.
- **Notes:** `+button` bindings send keynum and time as parameters for sub-frame accuracy. Key-up events always fire `CL_AddKeyUpCommands` regardless of catcher state.

### CL_CharEvent
- **Signature:** `void CL_CharEvent(int key)`
- **Purpose:** Dispatches shifted/capslocked printable characters to the active text handler.
- **Calls:** `Field_CharEvent`, `VM_Call`.

### Key_ClearStates
- **Signature:** `void Key_ClearStates(void)`
- **Purpose:** Synthesizes key-up events for all held keys and resets `down`/`repeats`; called on mode transitions.
- **Side effects:** Calls `CL_KeyEvent(i, qfalse, 0)` for each depressed key.

### CL_InitKeyCommands
- **Signature:** `void CL_InitKeyCommands(void)`
- **Purpose:** Registers `bind`, `unbind`, `unbindall`, `bindlist` with the command system.
- **Calls:** `Cmd_AddCommand`.

## Control Flow Notes
- **Init:** `CL_InitKeyCommands` is called during client startup to register commands.
- **Per-event:** The platform layer (`win_input.c` / `linux_joystick.c` etc.) calls `CL_KeyEvent` and `CL_CharEvent` as raw OS events arrive.
- **Render:** `Field_VariableSizeDraw` is called from console/chat draw code each frame.
- **Shutdown:** `Key_ClearStates` is called before mode transitions (e.g., `vid_restart`, disconnect).

## External Dependencies
- **Includes:** `client.h` → `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `Field_Clear`, `Field_CompleteCommand` (likely `cl_console.c`); `Con_PageUp/Down/Top/Bottom/ToggleConsole_f` (`cl_console.c`); `VM_Call` (`vm.c`); `Cbuf_AddText`, `Cmd_AddCommand`, `Cmd_Argc/Argv` (`cmd.c`); `Cvar_Set/VariableValue` (`cvar.c`); `Z_Free`, `CopyString` (memory); `FS_Printf` (`files.c`); `Sys_GetClipboardData` (platform); `SCR_Draw*` (`cl_scrn.c`); `CL_AddReliableCommand` (`cl_main.c`); `cvar_modifiedFlags` (`cvar.c`); `cls`, `clc`, `cgvm`, `uivm` (client globals).
