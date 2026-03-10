# code/qcommon/cmd.c

## File Purpose
Implements Quake III's command buffer and command execution system. It manages a text-based FIFO buffer of pending console commands, tokenizes command strings into arguments, and dispatches commands to registered handlers, cvars, game modules, or the server.

## Core Responsibilities
- Maintain a fixed-size circular-style command text buffer (`cmd_text`)
- Provide `Cbuf_*` API to append, insert, and execute buffered command text
- Tokenize raw command strings into argc/argv-style argument arrays
- Register and unload named command functions via a linked list
- Dispatch commands to: registered handlers → cvars → cgame → game → UI → server forward
- Implement built-in commands: `cmdlist`, `exec`, `vstr`, `echo`, `wait`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cmd_t` | struct | Ring-buffer descriptor: pointer to data, maxsize, cursize |
| `cmd_function_t` | struct | Linked-list node holding a command name and its `xcommand_t` callback |
| `xcommand_t` | typedef (`void (*)(void)`) | Callback type for all registered commands (defined in qcommon.h) |
| `cbufExec_t` | enum (in q_shared.h) | Execution mode: `EXEC_NOW`, `EXEC_INSERT`, `EXEC_APPEND` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cmd_wait` | `int` | global | Frame-delay counter; decremented by `Cbuf_Execute` each frame |
| `cmd_text` | `cmd_t` | global | Command buffer descriptor pointing into `cmd_text_buf` |
| `cmd_text_buf` | `byte[16384]` | global | Backing storage for the command buffer |
| `cmd_argc` | `int` | static | Argument count for current tokenized command |
| `cmd_argv` | `char*[MAX_STRING_TOKENS]` | static | Argument vector pointers into `cmd_tokenized` |
| `cmd_tokenized` | `char[BIG_INFO_STRING+MAX_STRING_TOKENS]` | static | Scratch buffer holding null-separated token strings |
| `cmd_cmd` | `char[BIG_INFO_STRING]` | static | Original unmodified command string (for rcon use) |
| `cmd_functions` | `cmd_function_t*` | static | Head of registered-commands linked list |

## Key Functions / Methods

### Cbuf_Init
- **Signature:** `void Cbuf_Init(void)`
- **Purpose:** Initializes the command buffer by pointing `cmd_text.data` at `cmd_text_buf`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes `cmd_text` global
- **Calls:** None
- **Notes:** Must be called before any `Cbuf_*` usage.

### Cbuf_AddText
- **Signature:** `void Cbuf_AddText(const char *text)`
- **Purpose:** Appends a command string to the tail of the buffer without adding `\n`.
- **Inputs:** `text` — null-terminated command string
- **Outputs/Return:** None
- **Side effects:** Modifies `cmd_text.data` and `cmd_text.cursize`; prints overflow warning
- **Calls:** `strlen`, `Com_Printf`, `Com_Memcpy`

### Cbuf_InsertText
- **Signature:** `void Cbuf_InsertText(const char *text)`
- **Purpose:** Inserts a command string at the head of the buffer (before pending commands), appending `\n`.
- **Inputs:** `text` — null-terminated command string
- **Outputs/Return:** None
- **Side effects:** Shifts existing buffer contents right; modifies `cmd_text`
- **Calls:** `strlen`, `Com_Printf`, `Com_Memcpy`
- **Notes:** Used by `exec` and `vstr` so inserted commands run before currently-buffered ones.

### Cbuf_Execute
- **Signature:** `void Cbuf_Execute(void)`
- **Purpose:** Drains the command buffer one line at a time, executing each via `Cmd_ExecuteString`. Respects `cmd_wait` for frame-delay.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `cmd_text`, decrements `cmd_wait`, calls `Cmd_ExecuteString`
- **Calls:** `Com_Memcpy`, `memmove`, `Cmd_ExecuteString`
- **Notes:** Called once per frame from the main loop. Commands inserted mid-execution (e.g., via `exec`) are processed in the same call since the buffer is compacted first.

### Cmd_TokenizeString
- **Signature:** `void Cmd_TokenizeString(const char *text_in)`
- **Purpose:** Parses a command string into tokens stored in `cmd_argc`/`cmd_argv`. Skips `//` and `/* */` comments; handles quoted strings.
- **Inputs:** `text_in` — raw command line
- **Outputs/Return:** None
- **Side effects:** Overwrites `cmd_argc`, `cmd_argv`, `cmd_tokenized`, `cmd_cmd`
- **Calls:** `Q_strncpyz`
- **Notes:** Does not handle `\"` escape sequences (noted with TTimo comment). Caps at `MAX_STRING_TOKENS`.

### Cmd_ExecuteString
- **Signature:** `void Cmd_ExecuteString(const char *text)`
- **Purpose:** Tokenizes a command line then dispatches to: registered command → cvar → cgame → game → UI → server forward.
- **Inputs:** `text` — command line string
- **Outputs/Return:** None
- **Side effects:** Mutates `cmd_functions` list order (MRU promotion); may call any registered handler or forward to server
- **Calls:** `Cmd_TokenizeString`, `Cmd_Argc`, `Q_stricmp`, `Cvar_Command`, `CL_GameCommand`, `SV_GameCommand`, `UI_GameCommand`, `CL_ForwardCommandToServer`
- **Notes:** Promotes matched command to head of list for cache-friendly future lookups.

### Cmd_AddCommand / Cmd_RemoveCommand
- **Purpose:** Register or unregister a named command in the `cmd_functions` linked list.
- **Notes:** `Cmd_AddCommand` allocates via `S_Malloc`/`CopyString`; `Cmd_RemoveCommand` frees via `Z_Free`. NULL function pointer registers a completion-only (server-forwarded) command.

### Cmd_Init
- **Signature:** `void Cmd_Init(void)`
- **Purpose:** Registers the five built-in commands: `cmdlist`, `exec`, `vstr`, `echo`, `wait`.
- **Calls:** `Cmd_AddCommand` ×5

## Control Flow Notes
- **Init:** `Cbuf_Init` → `Cmd_Init` called during `Com_Init`.
- **Per-frame:** `Com_Frame` calls `Cbuf_Execute`, which pulls lines from the buffer and calls `Cmd_ExecuteString` for each.
- **Input paths:** Key bindings and console input call `Cbuf_AddText`; `exec` and `vstr` use `Cbuf_InsertText` for priority insertion.

## External Dependencies
- `../game/q_shared.h` — `byte`, `qboolean`, `MAX_STRING_TOKENS`, `BIG_INFO_STRING`, `MAX_QPATH`, `cbufExec_t`, `Q_strncpyz`, `COM_DefaultExtension`, `va`
- `qcommon.h` — `xcommand_t`, `Com_Printf`, `Com_Error`, `Cvar_Command`, `Cvar_VariableString`, `FS_ReadFile`, `FS_FreeFile`, `S_Malloc`, `Z_Free`, `CopyString`, `Com_Filter`
- **Defined elsewhere:** `CL_GameCommand`, `SV_GameCommand`, `UI_GameCommand`, `CL_ForwardCommandToServer`, `com_cl_running`, `com_sv_running`
