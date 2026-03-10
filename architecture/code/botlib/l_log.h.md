# code/botlib/l_log.h

## File Purpose
Public header for the botlib logging subsystem. It declares the interface for opening, writing to, flushing, and closing a log file used by the bot library during development and debugging.

## Core Responsibilities
- Declare the log file lifecycle API (open, close, shutdown)
- Declare formatted write functions (with and without timestamps)
- Expose the underlying `FILE*` for external consumers
- Declare a flush function to force buffered output to disk

## Key Types / Data Structures
None.

## Global / File-Static State
None declared here; the backing `FILE*` is file-static in `l_log.c` and exposed only via `Log_FilePointer()`.

## Key Functions / Methods

### Log_Open
- Signature: `void Log_Open(char *filename);`
- Purpose: Opens a log file by name for subsequent writes.
- Inputs: `filename` — path to the log file to create/open.
- Outputs/Return: void
- Side effects: Allocates/opens a file handle stored in `l_log.c`'s internal state.
- Calls: Not inferable from this file.
- Notes: Must be called before any `Log_Write*` calls.

### Log_Close
- Signature: `void Log_Close(void);`
- Purpose: Closes the currently open log file.
- Inputs: None.
- Outputs/Return: void
- Side effects: Closes the file handle; subsequent writes are no-ops or errors.
- Calls: Not inferable from this file.
- Notes: Distinct from `Log_Shutdown`; implies a paired open/close pattern.

### Log_Shutdown
- Signature: `void Log_Shutdown(void);`
- Purpose: Closes the log file if one is currently open; safe to call unconditionally.
- Inputs: None.
- Outputs/Return: void
- Side effects: Conditionally closes the file handle.
- Notes: Intended for use at botlib teardown to guarantee cleanup.

### Log_Write
- Signature: `void QDECL Log_Write(char *fmt, ...);`
- Purpose: Writes a `printf`-style formatted string to the open log file.
- Inputs: `fmt` — format string; variadic arguments.
- Outputs/Return: void
- Side effects: I/O to the log file.
- Notes: `QDECL` enforces the calling convention required for variadic functions on Windows (typically `__cdecl`).

### Log_WriteTimeStamped
- Signature: `void QDECL Log_WriteTimeStamped(char *fmt, ...);`
- Purpose: Same as `Log_Write` but prepends a time stamp to each entry.
- Inputs: `fmt` — format string; variadic arguments.
- Outputs/Return: void
- Side effects: I/O to the log file; reads system time.
- Notes: Useful for correlating log entries with game time or wall-clock time.

### Log_FilePointer
- Signature: `FILE *Log_FilePointer(void);`
- Purpose: Returns the raw `FILE*` for the currently open log file.
- Inputs: None.
- Outputs/Return: Pointer to the open `FILE`, or `NULL` if none is open.
- Side effects: None.
- Notes: Allows callers to perform direct `fprintf`/`fwrite` if needed; breaks encapsulation, so use sparingly.

### Log_Flush
- Signature: `void Log_Flush(void);`
- Purpose: Flushes the log file's write buffer to disk.
- Inputs: None.
- Outputs/Return: void
- Side effects: Calls `fflush` (or equivalent) on the internal `FILE*`.
- Notes: Useful before a potential crash or assert to ensure the last log entries are not lost.

## Control Flow Notes
This header is consumed by botlib subsystems that need diagnostic logging. `Log_Open` is called during botlib initialization, `Log_Shutdown` during teardown. `Log_Write`/`Log_WriteTimeStamped` are called throughout the frame at arbitrary points; `Log_Flush` may be called on demand or periodically.

## External Dependencies
- `<stdio.h>` — for the `FILE` type used in `Log_FilePointer`.
- `QDECL` — macro defined in `q_shared.h` (typically expands to `__cdecl` on Windows, empty on others).
- Implementation defined in `code/botlib/l_log.c`.
