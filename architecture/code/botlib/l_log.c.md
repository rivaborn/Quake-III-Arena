# code/botlib/l_log.c

## File Purpose
Provides a simple file-based logging facility for the botlib subsystem. It manages a single global log file, supporting plain and timestamped write operations gated by the `"log"` library variable.

## Core Responsibilities
- Open and close a single log file on demand, guarded by the `"log"` libvar
- Write formatted (variadic) messages to the log file
- Write timestamped, sequenced entries using `botlibglobals.time`
- Flush the log file buffer on demand
- Expose the raw `FILE*` pointer for external direct writes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `logfile_t` | struct | Holds the log file state: filename, `FILE*` pointer, and write counter |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `logfile` | `static logfile_t` | static (file) | Single global log file instance; tracks open file handle, filename, and write count |

## Key Functions / Methods

### Log_Open
- **Signature:** `void Log_Open(char *filename)`
- **Purpose:** Opens the log file for binary writing if the `"log"` libvar is non-zero and no file is already open.
- **Inputs:** `filename` — path to the log file to create/truncate.
- **Outputs/Return:** None.
- **Side effects:** Writes to `logfile.fp` and `logfile.filename`; prints messages via `botimport.Print`.
- **Calls:** `LibVarValue`, `fopen`, `strncpy`, `botimport.Print`
- **Notes:** Opens in `"wb"` mode (binary write, truncates existing). Silently returns if `"log"` is `"0"`. Errors if a log is already open.

### Log_Close
- **Signature:** `void Log_Close(void)`
- **Purpose:** Closes the currently open log file and nulls the handle.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `fclose`; nulls `logfile.fp`; prints via `botimport.Print`.
- **Calls:** `fclose`, `botimport.Print`
- **Notes:** No-op if no file is open.

### Log_Shutdown
- **Signature:** `void Log_Shutdown(void)`
- **Purpose:** Shuts down logging; delegates to `Log_Close` if a file is open.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Transitively closes the log file.
- **Calls:** `Log_Close`

### Log_Write
- **Signature:** `void QDECL Log_Write(char *fmt, ...)`
- **Purpose:** Writes a variadic formatted string to the log file without a newline or timestamp. Flushes after each write.
- **Inputs:** `fmt` — printf-style format string and variadic args.
- **Outputs/Return:** None.
- **Side effects:** Writes to `logfile.fp`; calls `fflush`.
- **Calls:** `vfprintf`, `fflush`
- **Notes:** No-op if file is not open.

### Log_WriteTimeStamped
- **Signature:** `void QDECL Log_WriteTimeStamped(char *fmt, ...)`
- **Purpose:** Writes a sequenced, time-stamped formatted entry (`write_number  HH:MM:SS:cs  <message>\r\n`) and increments the write counter.
- **Inputs:** `fmt` — printf-style format string and variadic args.
- **Outputs/Return:** None.
- **Side effects:** Writes to `logfile.fp`; increments `logfile.numwrites`; calls `fflush`. Reads `botlibglobals.time` for timestamp.
- **Calls:** `fprintf`, `vfprintf`, `fflush`
- **Notes:** Time is decomposed into hours/minutes/seconds/centiseconds from the float `botlibglobals.time`. Centiseconds calculation uses integer truncation arithmetic.

### Log_FilePointer
- **Signature:** `FILE *Log_FilePointer(void)`
- **Purpose:** Returns the raw `FILE*` of the open log for external use.
- **Inputs:** None.
- **Outputs/Return:** `logfile.fp` (may be `NULL` if not open).
- **Side effects:** None.
- **Calls:** None.

### Log_Flush
- **Signature:** `void Log_Flush(void)`
- **Purpose:** Flushes the log file's I/O buffer without closing it.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `fflush(logfile.fp)`.
- **Calls:** `fflush`

## Control Flow Notes
This file has no frame/update loop participation of its own. `Log_Open` is called during botlib setup (controlled by the `"log"` libvar); `Log_Shutdown` is called during botlib teardown. `Log_Write` / `Log_WriteTimeStamped` are called ad hoc by other botlib modules throughout the session.

## External Dependencies
- `<stdlib.h>`, `<stdio.h>`, `<string.h>` — standard C I/O and string functions
- `../game/q_shared.h` — shared engine types (`QDECL`, etc.)
- `../game/botlib.h` — `PRT_MESSAGE`, `PRT_ERROR` print type constants
- `be_interface.h` — `botimport` (for `botimport.Print`) and `botlibglobals` (for `botlibglobals.time`)
- `l_libvar.h` — `LibVarValue` (defined in `l_libvar.c`)
- `botimport.Print` — defined elsewhere (host engine), called via function pointer
- `botlibglobals` — defined in `be_interface.c`
