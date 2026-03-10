# code/win32/win_main.c

## File Purpose
Win32-specific platform entry point and system abstraction layer for Quake III Arena. It implements `WinMain`, the OS event loop, file system enumeration, DLL loading, and miscellaneous system services required by the engine's `Sys_*` API contract.

## Core Responsibilities
- Owns `WinMain` and the top-level game loop (`IN_Frame` → `Com_Frame`)
- Queues and dispatches system events (keyboard, mouse, network, console) via `Sys_QueEvent` / `Sys_GetEvent`
- Provides Win32 filesystem services: directory listing, filtered file enumeration, `Sys_Mkdir`, `Sys_Cwd`
- Loads and unloads game module DLLs (`Sys_LoadDll` / `Sys_UnloadDll`) with a security warning in release builds
- Implements stub background file streaming (`Sys_InitStreamThread` etc.) — full threaded implementation is `#if 0`'d out
- Initializes the system: high-resolution timer, OS version detection, CPU identification, input subsystem
- Handles fatal error and clean-quit paths (`Sys_Error`, `Sys_Quit`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `WinVars_t` | struct (defined in `win_local.h`) | Global Win32 window/instance state: `hWnd`, `hInstance`, `osversion`, `sysMsgTime`, `isMinimized` |
| `sysEvent_t` | struct (defined in `qcommon.h`) | Timestamped engine event: type, two int values, optional heap pointer |
| `streamsIO_t` | struct (disabled `#if 0`) | Per-file streaming buffer state for the threaded read-ahead path |
| `streamState_t` | struct (disabled `#if 0`) | Thread handle + per-file `streamsIO_t` array for background streaming |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sys_cmdline` | `char[MAX_STRING_CHARS]` | file-static | Stores raw command-line string from `WinMain` |
| `eventQue` | `sysEvent_t[256]` | global | Circular ring buffer of pending system events |
| `eventHead` | `int` | global | Write index into `eventQue` |
| `eventTail` | `int` | global | Read index into `eventQue` |
| `sys_packetReceived` | `byte[MAX_MSGLEN]` | global | Scratch buffer for inbound network packets |
| `totalMsec` / `countMsec` | `int` | global | Accumulated frame timing statistics |
| `fh` | `int` | global (ALT_SPANK) | File descriptor for debug spank-log output |

## Key Functions / Methods

### WinMain
- **Signature:** `int WINAPI WinMain(HINSTANCE hInstance, HINSTANCE hPrevInstance, LPSTR lpCmdLine, int nCmdShow)`
- **Purpose:** OS entry point; bootstraps all engine subsystems, then runs the infinite game loop.
- **Inputs:** Win32 instance handles and command-line string.
- **Outputs/Return:** `int` (process exit code; loop never exits normally).
- **Side effects:** Creates console window, initializes timer, calls `Com_Init`, `NET_Init`, enters infinite loop.
- **Calls:** `Sys_CreateConsole`, `SetErrorMode`, `Sys_Milliseconds`, `Sys_InitStreamThread`, `Com_Init`, `NET_Init`, `Sys_ShowConsole`, `IN_Frame`, `Com_Frame`.
- **Notes:** `hPrevInstance` is always NULL on Win32/Win64; guarded with early return.

### Sys_Init
- **Signature:** `void Sys_Init(void)`
- **Purpose:** Post-common init: sets timer resolution, detects OS version and CPU, registers input commands, initializes input.
- **Inputs:** None (reads `g_wv.osversion`, cvars).
- **Outputs/Return:** void.
- **Side effects:** Calls `timeBeginPeriod(1)`, sets `arch`/`sys_cpustring`/`sys_cpuid`/`username` cvars, calls `IN_Init`.
- **Calls:** `Cmd_AddCommand`, `GetVersionEx`, `Sys_Error`, `Cvar_Set`, `Cvar_Get`, `Cvar_SetValue`, `Sys_GetProcessorId`, `Sys_GetCurrentUser`, `IN_Init`.

### Sys_GetEvent
- **Signature:** `sysEvent_t Sys_GetEvent(void)`
- **Purpose:** Returns the next system event; pumps the Win32 message queue and polls console/network if the ring buffer is empty.
- **Inputs:** None.
- **Outputs/Return:** One `sysEvent_t`; empty event with current time if nothing pending.
- **Side effects:** May call `Com_Quit_f` on `WM_QUIT`; allocates heap for console and packet events.
- **Calls:** `PeekMessage`, `GetMessage`, `TranslateMessage`, `DispatchMessage`, `Sys_ConsoleInput`, `MSG_Init`, `Sys_GetPacket`, `Sys_QueEvent`, `Z_Malloc`, `timeGetTime`.

### Sys_QueEvent
- **Signature:** `void Sys_QueEvent(int time, sysEventType_t type, int value, int value2, int ptrLength, void *ptr)`
- **Purpose:** Pushes an event into the ring buffer; drops oldest and frees its pointer on overflow.
- **Side effects:** Modifies `eventQue`, `eventHead`, `eventTail`; may call `Z_Free`.

### Sys_LoadDll
- **Signature:** `void * QDECL Sys_LoadDll(const char *name, char *fqpath, int (QDECL **entryPoint)(int,...), int (QDECL *systemcalls)(int,...))`
- **Purpose:** Loads a game-module DLL, resolves `dllEntry` and `vmMain`, and calls `dllEntry(systemcalls)` to wire the syscall table.
- **Inputs:** Module base name, output path buffer, pointers to entry-point and syscall function pointers.
- **Outputs/Return:** `HINSTANCE` on success, `NULL` on failure; fills `fqpath` and `*entryPoint`.
- **Side effects:** Shows a `MessageBoxEx` security warning in release builds; calls `dllEntry`.
- **Calls:** `LoadLibrary`, `GetProcAddress`, `FreeLibrary`, `FS_BuildOSPath`, `Cvar_VariableString`, `Cvar_VariableIntegerValue`, `FS_FileExists`, `MessageBoxEx`.

### Sys_ListFiles / Sys_ListFilteredFiles
- **Purpose:** Enumerate files in a directory using `_findfirst`/`_findnext`; supports extension filter or glob pattern; bubble-sorts results.
- **Notes:** Allocates per-entry strings via `CopyString`; caller must free with `Sys_FreeFileList`.

### Sys_Error
- **Signature:** `void QDECL Sys_Error(const char *error, ...)`
- **Purpose:** Fatal error handler — displays message in console window and spins pumping messages until quit.
- **Side effects:** Calls `IN_Shutdown`, `timeEndPeriod`, shows console; never returns.

## Control Flow Notes

`WinMain` is the init/frame/shutdown axis:
- **Init:** console → `Com_Init` → `NET_Init` → (later `Sys_Init` called from `Com_Init` chain)
- **Frame:** `IN_Frame` then `Com_Frame` each iteration; sleeps 5 ms if minimized or dedicated
- **Shutdown:** `Sys_Error`/`Sys_Quit` call `exit()`; normal loop exit never reached

`Sys_GetEvent` is polled by `Com_EventLoop` inside `Com_Frame`.

## External Dependencies

- `../client/client.h` — `IN_Frame`, `IN_Init`, `IN_Shutdown`
- `../qcommon/qcommon.h` — `Com_Init`, `Com_Frame`, `NET_Init`, `NET_Restart`, `Sys_Milliseconds`, `Z_Malloc`, `Z_Free`, `FS_Read`, `FS_Seek`, `Cvar_*`, `Cmd_AddCommand`, `MSG_Init`
- `win_local.h` — `WinVars_t g_wv`, `Sys_CreateConsole`, `Sys_DestroyConsole`, `Sys_ConsoleInput`, `Sys_GetPacket`, `MainWndProc`, `Conbuf_AppendText`, `Sys_ShowConsole`, `Sys_SetErrorText`
- Win32 API: `<windows.h>`, `timeBeginPeriod`/`timeEndPeriod`/`timeGetTime` (winmm), `LoadLibrary`, `GetProcAddress`, `FreeLibrary`, `GlobalMemoryStatus`, `_findfirst`/`_findnext`/`_findclose`
- **Defined elsewhere:** `FS_BuildOSPath`, `Sys_GetProcessorId`, `Sys_GetCurrentUser`, `Sys_Milliseconds`, `CopyString`, `Q_strncpyz`, `Com_sprintf`, `Com_FilterPath`
