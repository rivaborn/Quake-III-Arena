# code/bspc/l_log.h

## File Purpose
Public header for a simple logging utility used by the BSPC (BSP Compiler) tool. It declares the interface for opening, writing to, timestamping, and closing a log file, with an optional Windows-specific print hook.

## Core Responsibilities
- Declare the log file lifecycle API (open, close, shutdown)
- Declare formatted write functions (stdout+file, file-only, timestamped)
- Expose raw `FILE*` access for external consumers
- Provide a flush mechanism for the log file
- Conditionally declare a Windows GUI print callback (`WINBSPC`)

## Key Types / Data Structures
None.

## Global / File-Static State
None declared here; the backing `FILE*` is encapsulated in the corresponding `.c` implementation.

## Key Functions / Methods

### Log_Open
- Signature: `void Log_Open(char *filename)`
- Purpose: Opens a log file by name for writing.
- Inputs: `filename` — path to the log file.
- Outputs/Return: void
- Side effects: Allocates/opens a file handle stored in the `.c` implementation's file-static state.
- Calls: Not inferable from this file.
- Notes: Must be called before any `Log_Write*` calls.

### Log_Close
- Signature: `void Log_Close(void)`
- Purpose: Closes the currently open log file.
- Inputs: None
- Outputs/Return: void
- Side effects: Closes and nulls the internal file handle.
- Calls: Not inferable from this file.
- Notes: Distinguished from `Log_Shutdown`; likely does not guard against a null handle.

### Log_Shutdown
- Signature: `void Log_Shutdown(void)`
- Purpose: Closes the log file if one is currently open; safe to call unconditionally.
- Inputs: None
- Outputs/Return: void
- Side effects: Conditionally closes the internal file handle.
- Calls: Likely calls `Log_Close` internally.
- Notes: Intended as the teardown entry point at program exit.

### Log_Print
- Signature: `void Log_Print(char *fmt, ...)`
- Purpose: Prints a formatted message to both stdout and the open log file.
- Inputs: `fmt` — printf-style format string; variadic arguments.
- Outputs/Return: void
- Side effects: I/O to stdout and log file.
- Calls: Not inferable from this file.

### Log_Write
- Signature: `void Log_Write(char *fmt, ...)`
- Purpose: Writes a formatted message exclusively to the open log file (no stdout).
- Inputs: `fmt` — printf-style format string; variadic arguments.
- Outputs/Return: void
- Side effects: I/O to log file only.
- Calls: Not inferable from this file.

### Log_WriteTimeStamped
- Signature: `void Log_WriteTimeStamped(char *fmt, ...)`
- Purpose: Writes a formatted message prefixed with a timestamp to the log file.
- Inputs: `fmt` — printf-style format string; variadic arguments.
- Outputs/Return: void
- Side effects: I/O to log file; reads system time.
- Calls: Not inferable from this file.

### Log_FileStruct
- Signature: `FILE *Log_FileStruct(void)`
- Purpose: Returns the raw `FILE*` handle for the currently open log file.
- Inputs: None
- Outputs/Return: Pointer to the internal `FILE` structure, or `NULL` if not open.
- Side effects: None.
- Notes: Allows callers to perform direct `fprintf`/`fwrite` if needed.

### Log_Flush
- Signature: `void Log_Flush(void)`
- Purpose: Flushes the log file's I/O buffer to disk.
- Inputs: None
- Outputs/Return: void
- Side effects: Calls `fflush` on the internal file handle.

### WinBSPCPrint *(conditional)*
- Signature: `void WinBSPCPrint(char *str)`
- Purpose: Windows-specific callback to route log output to the WinBSPC GUI console.
- Inputs: `str` — null-terminated string to display.
- Outputs/Return: void
- Side effects: Platform GUI I/O.
- Notes: Only compiled when `WINBSPC` is defined.

## Control Flow Notes
This is a pure header; it participates in BSPC tool initialization (log opened early in `main`/startup), used throughout compilation passes for diagnostics, and closed during shutdown. Not part of the runtime game engine frame loop.

## External Dependencies
- `<stdio.h>` — `FILE*` type (implicitly required by `Log_FileStruct`; must be included before this header in translation units).
- `WINBSPC` — preprocessor symbol controlling the Windows GUI variant; defined elsewhere in the build system.
- Implementation defined in `code/bspc/l_log.c`.
