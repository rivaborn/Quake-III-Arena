# code/null/null_main.c

## File Purpose
A minimal null/stub system driver for Quake III Arena, intended to aid porting efforts to new platforms. It provides no-op or trivially forwarding implementations of all required `Sys_*` platform abstraction functions, and contains the program entry point.

## Core Responsibilities
- Provide a compilable stub for all `Sys_*` platform interface functions required by `qcommon`
- Implement the program entry point (`main`) that initializes the engine and runs the main loop
- Forward streamed file I/O to standard C `fread`/`fseek`
- Print fatal errors to stdout and terminate the process
- Serve as a minimal baseline for porting to platforms without a real system driver

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sys_curtime` | `int` | global | Tracks current system time; unused/unset in this null driver |

## Key Functions / Methods

### main
- **Signature:** `void main(int argc, char **argv)`
- **Purpose:** Program entry point; initializes the common engine layer and spins in the main frame loop forever.
- **Inputs:** `argc`, `argv` — standard command-line arguments passed directly to `Com_Init`
- **Outputs/Return:** Never returns (infinite loop or `exit`)
- **Side effects:** Calls `Com_Init` (initializes all engine subsystems), then calls `Com_Frame` in a tight infinite loop
- **Calls:** `Com_Init`, `Com_Frame`
- **Notes:** No graceful shutdown path; relies on `Sys_Quit` or `Sys_Error` to terminate

### Sys_Error
- **Signature:** `void Sys_Error(char *error, ...)`
- **Purpose:** Fatal error handler; prints a formatted message to stdout and calls `exit(1)`
- **Inputs:** `error` — printf-style format string; variadic arguments
- **Outputs/Return:** Does not return
- **Side effects:** Writes to stdout; terminates the process
- **Calls:** `printf`, `vprintf`, `exit`
- **Notes:** Implements the platform-required `Sys_Error` contract

### Sys_Quit
- **Signature:** `void Sys_Quit(void)`
- **Purpose:** Clean shutdown; immediately exits with code 0
- **Side effects:** Terminates the process via `exit(0)`

### Sys_StreamedRead
- **Signature:** `int Sys_StreamedRead(void *buffer, int size, int count, FILE *f)`
- **Purpose:** Reads data from a streamed file; delegates directly to `fread`
- **Inputs:** Standard `fread` parameters
- **Outputs/Return:** Number of items read
- **Notes:** `Sys_BeginStreamedFile` and `Sys_EndStreamedFile` are no-ops; no actual async read-ahead is implemented

### Sys_StreamSeek
- **Signature:** `void Sys_StreamSeek(FILE *f, int offset, int origin)`
- **Purpose:** Seeks within a streamed file; delegates directly to `fseek`

### Sys_EarlyOutput
- **Signature:** `void Sys_EarlyOutput(char *string)`
- **Purpose:** Outputs a string to stdout before the full console is available
- **Calls:** `printf`

### Notes
- `Sys_GetGameAPI`, `Sys_GetClipboardData`, `Sys_FindFirst`, `Sys_FindNext` all return `NULL` — DLL/game loading and file enumeration are unsupported in the null driver.
- `Sys_Milliseconds`, `Sys_Mkdir`, `Sys_FindClose`, `Sys_Init`, `Sys_UnloadGame` are all no-ops or return zero.
- The `FILE *` signature on stream functions differs from `qcommon.h`'s `fileHandle_t` declarations — this file predates or diverges from the abstracted FS handle API.

## Control Flow Notes
This file owns the program entry point. On startup: `main` → `Com_Init` (engine init) → infinite loop calling `Com_Frame` each iteration. Abnormal termination flows through `Sys_Error` → `exit(1)` or `Sys_Quit` → `exit(0)`. There is no render, audio, or input integration; all such stubs are no-ops.

## External Dependencies
- `<errno.h>`, `<stdio.h>` — standard C I/O and error codes
- `../qcommon/qcommon.h` — engine-wide common declarations; defines `Com_Init`, `Com_Frame`, and the full `Sys_*` interface contract
- **Defined elsewhere:** `Com_Init`, `Com_Frame` (in `qcommon/common.c`); all `Sys_*` signatures are declared in `qcommon.h` but the authoritative platform implementations live in `code/win32/`, `code/unix/`, `code/macosx/`
