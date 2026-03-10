# code/unix/unix_main.c

## File Purpose
This is the Linux/Unix platform entry point for Quake III Arena, implementing the OS-level system layer. It provides the `main()` function, the event loop, DLL loading, TTY console I/O, and all `Sys_*` functions required by the engine's platform abstraction.

## Core Responsibilities
- Houses `main()`: parses args, initializes engine via `Com_Init`, and runs the main `Com_Frame` loop
- Implements the system event queue (`Sys_QueEvent` / `Sys_GetEvent`) feeding input, console, and network events
- Provides TTY console with raw-mode input, line editing, tab completion, and command history
- Implements `Sys_LoadDll` / `Sys_UnloadDll` for native game/cgame/ui module loading via `dlopen`
- Implements `Sys_Error`, `Sys_Quit`, `Sys_Exit` — the unified shutdown/error paths
- Provides no-op or pass-through background file streaming stubs
- Configures architecture cvar and FPU state at startup

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `field_t` | typedef struct (from qcommon.h) | Editable text field; used for tty console input line |
| `sysEvent_t` | typedef struct (from qcommon.h) | System event (key, mouse, console, packet) placed on the event queue |
| `streamState_t` | struct (disabled `#else` branch) | Buffered async file streaming state; compiled out in active build |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `re` | `refexport_t` | global | Exported functions from the renderer DLL |
| `sys_frame_time` | `unsigned` | global | Current frame timestamp (set externally) |
| `saved_euid` | `uid_t` | global | Saved effective UID, used to drop privileges at startup |
| `stdin_active` | `qboolean` | global | Whether stdin is still producing input (dedicated server) |
| `ttycon` | `cvar_t *` | static | `ttycon` cvar controlling TTY mode |
| `ttycon_on` | `qboolean` | static | Whether TTY raw mode is active |
| `ttycon_hide` | `int` | static | Nesting counter for suppressing tty line display during prints |
| `tty_erase`, `tty_eof` | `int` | static | Terminal erase/EOF control characters |
| `tty_tc` | `struct termios` | static | Saved original terminal attributes for restore on shutdown |
| `tty_con` | `field_t` | static | Current tty input line being edited |
| `ttyEditLines[TTY_HISTORY]` | `field_t[32]` | static | TTY command history ring buffer |
| `hist_current`, `hist_count` | `int` | static | History navigation cursor and count |
| `eventQue[MAX_QUED_EVENTS]` | `sysEvent_t[256]` | global | Circular event queue |
| `eventHead`, `eventTail` | `int` | global | Head/tail indices into `eventQue` |
| `sys_packetReceived` | `byte[MAX_MSGLEN]` | global | Scratch buffer for incoming network packets |

## Key Functions / Methods

### main
- **Signature:** `int main(int argc, char* argv[])`
- **Purpose:** Platform entry point; drops privileges, merges argv into a command line string, initializes engine and network, starts the TTY console, then loops forever calling `Com_Frame`.
- **Inputs:** `argc`, `argv`
- **Outputs/Return:** Never returns normally.
- **Side effects:** Calls `seteuid(getuid())` to drop SUID privilege. Allocates `cmdline` with `malloc` (never freed — intentional). Sets stdin to non-blocking via `fcntl`. Calls `Com_Init`, `NET_Init`, `Sys_ConsoleInputInit`, then enters infinite loop.
- **Calls:** `Sys_ParseArgs`, `Sys_SetDefaultCDPath`, `Com_Init`, `NET_Init`, `Sys_ConsoleInputInit`, `fcntl`, `InitSig` (dedicated only), `Sys_ConfigureFPU`, `Com_Frame`
- **Notes:** Event queues are zero-initialized before `Com_Init`. `InitSig` is only called in `DEDICATED` builds here; non-dedicated builds rely on GLimp to call it.

### Sys_GetEvent
- **Signature:** `sysEvent_t Sys_GetEvent(void)`
- **Purpose:** Drains the event queue; if empty, pumps all input sources (keyboard, console, joystick, network) and queues new events before returning one.
- **Inputs:** None
- **Outputs/Return:** One `sysEvent_t`; returns a zeroed event with current time if nothing available.
- **Side effects:** Calls `Z_Malloc` for console string and network packet copies placed into `eventQue`. Advances `eventTail`.
- **Calls:** `Sys_SendKeyEvents`, `Sys_ConsoleInput`, `Z_Malloc`, `Sys_QueEvent`, `IN_Frame`, `MSG_Init`, `Sys_GetPacket`

### Sys_QueEvent
- **Signature:** `void Sys_QueEvent(int time, sysEventType_t type, int value, int value2, int ptrLength, void *ptr)`
- **Purpose:** Appends one event to the circular queue. Discards (and frees evPtr) oldest event on overflow.
- **Inputs:** Timestamp (0 = current time), event type, two integer values, optional heap pointer and its length.
- **Side effects:** Modifies `eventHead`/`eventTail`. Calls `Z_Free` on overflow. Calls `Sys_Milliseconds` if `time == 0`.

### Sys_LoadDll
- **Signature:** `void *Sys_LoadDll(const char *name, char *fqpath, int (**entryPoint)(int,...), int (*systemcalls)(int,...))`
- **Purpose:** Loads a native game module (.so) by searching pwdpath → homepath → basepath in order. Resolves `dllEntry` and `vmMain` symbols, calls `dllEntry(systemcalls)` to wire up the syscall table.
- **Inputs:** Bare module name, output path buffer, output `vmMain` function pointer, engine syscall dispatcher.
- **Outputs/Return:** `dlopen` handle on success, `NULL` on failure. Writes fully-qualified path into `fqpath`.
- **Side effects:** `dlopen`/`dlsym` calls. Calls `dllEntry` which registers engine callbacks into the loaded module. Logs extensively via `Com_Printf`.
- **Calls:** `Sys_Cwd`, `Cvar_VariableString`, `FS_BuildOSPath`, `dlopen`, `dlsym`, `dlclose`, `Com_Printf`, `Com_Error`, `Q_strncpyz`
- **Notes:** Architecture suffix (i386/ppc/axp/mips) is baked in at compile time via `#if`.

### Sys_ConsoleInput
- **Signature:** `char *Sys_ConsoleInput(void)`
- **Purpose:** Non-blocking read of one character from stdin. In tty mode: handles backspace, tab-complete, VT100 arrow keys (history nav), and echoes printable characters. In non-tty mode: `select`-based line read for dedicated server.
- **Outputs/Return:** Pointer to completed line string, or `NULL` if not yet complete.
- **Side effects:** Writes echo characters to stdout (fd 1) via `write`. Modifies `tty_con`. Calls `Hist_Add`, `Field_CompleteCommand`.

### Sys_ConsoleInputInit / Sys_ConsoleInputShutdown
- **Purpose:** `Init` puts stdin into raw mode (no echo, no canonical) if `ttycon` is set and stdin is a tty. `Shutdown` restores saved `tty_tc` terminal attributes.
- **Side effects:** `tcsetattr`/`tcgetattr` on fd 0. Installs `SIG_IGN` for `SIGTTIN`/`SIGTTOU`.

### Sys_Error
- **Signature:** `void Sys_Error(const char *error, ...)`
- **Purpose:** Fatal error handler. Shuts down client, prints to stderr, calls `Sys_Exit(1)`.
- **Side effects:** `CL_Shutdown`, `tty_Hide`, `fcntl` to restore blocking stdin.

### Sys_Exit
- **Signature:** `void Sys_Exit(int ex)`
- **Purpose:** Single exit point. Release builds call `_exit` (bypasses atexit, avoids GL DLL atexit issues); debug builds `assert(ex==0)` then `exit`.

## Control Flow Notes
- `main()` is the engine entry point. After initialization it enters an infinite `while(1) { Com_Frame(); }` loop — there is no graceful return.
- `Com_Frame` calls back into this file via `Sys_GetEvent` each frame to drain platform input.
- `Sys_LoadDll` is called during VM initialization (`vm.c`) to load cgame/game/ui as native shared libraries, wiring their `vmMain` entry point and syscall table.
- Shutdown flows through `Sys_Quit` → `CL_Shutdown` → `Sys_Exit`, or `Sys_Error` → `Sys_Exit` on fatal errors.

## External Dependencies
- **Includes:** `<dlfcn.h>`, `<termios.h>`, `<sys/time.h>`, `<signal.h>`, `<mntent.h>` (Linux), `<fpu_control.h>` (Linux i386)
- **Defined elsewhere:** `Com_Init`, `Com_Frame`, `NET_Init`, `CL_Shutdown`, `IN_Init/Shutdown/Frame`, `Sys_SendKeyEvents`, `Sys_GetPacket`, `Sys_Milliseconds`, `FS_BuildOSPath`, `FS_Read`, `FS_Seek`, `Field_Clear`, `Field_CompleteCommand`, `Z_Malloc`, `Z_Free`, `Cvar_Get`, `Cvar_Set`, `Cvar_VariableString`, `Cmd_AddCommand`, `MSG_Init`, `InitSig`, `Sys_Cwd`, `Sys_SetDefaultCDPath`, `Sys_GetCurrentUser`
