# code/bspc/l_log.c

## File Purpose
Provides a simple file-based logging facility for the BSPC (BSP Compiler) tool. It manages a single global log file with functions to open, close, print, and flush log output, with optional console mirroring controlled by a `verbose` flag.

## Core Responsibilities
- Open and close a single global log file by filename
- Write formatted messages to the log file (with and without console mirroring)
- Normalize line endings to `\r\n` (CRLF) before writing to file
- Flush the log file on demand and after every write
- Provide access to the underlying `FILE*` handle for external use

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `logfile_t` | struct | Holds log file state: filename buffer, `FILE*` pointer, and write count |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `logfile` | `logfile_t` | global (file-static by convention) | Singleton log file instance; all log functions operate on this |

## Key Functions / Methods

### Log_Open
- **Signature:** `void Log_Open(char *filename)`
- **Purpose:** Opens the log file for binary write (`"wb"`), storing the filename.
- **Inputs:** `filename` — path string to the desired log file.
- **Outputs/Return:** None. Prints status to `stdout`.
- **Side effects:** Sets `logfile.fp` and copies `filename` into `logfile.filename`.
- **Calls:** `fopen`, `strncpy`, `printf`, `strlen`
- **Notes:** Guards against NULL/empty filename and double-open. Opens in `"wb"` mode, so CRLF normalization in `Log_UnifyEndOfLine` is the sole mechanism for Windows line endings.

### Log_Close
- **Signature:** `void Log_Close(void)`
- **Purpose:** Closes the open log file and NULLs the `FILE*`.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Sets `logfile.fp = NULL`.
- **Calls:** `fclose`, `printf`
- **Notes:** Prints an error to stdout if `fclose` fails but does not retry.

### Log_Shutdown
- **Signature:** `void Log_Shutdown(void)`
- **Purpose:** Safe shutdown wrapper; closes the log only if it is open.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Delegates to `Log_Close`.
- **Calls:** `Log_Close`

### Log_UnifyEndOfLine
- **Signature:** `void Log_UnifyEndOfLine(char *buf)`
- **Purpose:** Converts bare `\n` to `\r\n` in-place by shifting bytes rightward with `memmove`.
- **Inputs:** `buf` — null-terminated string to modify in place.
- **Outputs/Return:** None (mutates `buf`).
- **Side effects:** May expand buffer content; caller must ensure sufficient allocation.
- **Calls:** `memmove`, `strlen`
- **Notes:** Risk of buffer overrun if input is near the caller's 2048-byte stack buffer and contains many bare newlines.

### Log_Print
- **Signature:** `void Log_Print(char *fmt, ...)`
- **Purpose:** Formats a message, optionally prints to console (if `verbose`), and writes to log file if open.
- **Inputs:** `fmt` — printf-style format string and variadic args.
- **Outputs/Return:** None.
- **Side effects:** Console I/O; file I/O with immediate flush.
- **Calls:** `vsprintf`, `printf`, `WinBSPCPrint` (Win only), `Log_UnifyEndOfLine`, `fprintf`, `fflush`
- **Notes:** Uses a fixed 2048-byte stack buffer — no bounds checking on `vsprintf`.

### Log_Write
- **Signature:** `void Log_Write(char *fmt, ...)`
- **Purpose:** Writes a formatted message to the log file only (no console output). No-ops if log is not open.
- **Inputs:** `fmt` — printf-style format string and variadic args.
- **Outputs/Return:** None.
- **Side effects:** File I/O with immediate flush.
- **Calls:** `vsprintf`, `Log_UnifyEndOfLine`, `fprintf`, `fflush`

### Log_WriteTimeStamped
- **Signature:** `void Log_WriteTimeStamped(char *fmt, ...)`
- **Purpose:** Writes a formatted message to the log and increments `logfile.numwrites`. Timestamp code is commented out.
- **Inputs:** `fmt` — printf-style format string and variadic args.
- **Outputs/Return:** None.
- **Side effects:** Increments `logfile.numwrites`; file I/O with flush.
- **Calls:** `vfprintf`, `fflush`
- **Notes:** Unlike `Log_Write`, passes args directly to `vfprintf` without CRLF normalization.

### Log_FileStruct
- **Signature:** `FILE *Log_FileStruct(void)`
- **Purpose:** Returns the raw `FILE*` for the open log file.
- **Outputs/Return:** `logfile.fp` or `NULL` if not open.

### Log_Flush
- **Signature:** `void Log_Flush(void)`
- **Purpose:** Explicitly flushes the log file buffer.
- **Calls:** `fflush`

## Control Flow Notes
This file has no frame/update/render lifecycle. It is utility infrastructure called at BSPC tool startup (`Log_Open`), throughout processing (`Log_Print`, `Log_Write`), and at shutdown (`Log_Shutdown`). It is not involved in game runtime.

## External Dependencies
- `<stdlib.h>`, `<stdio.h>`, `<string.h>` — standard C I/O and string utilities
- `qbsp.h` — pulls in `verbose` (extern global) and the `WinBSPCPrint` declaration (Windows GUI build only)
- `verbose` — extern boolean controlling console mirroring; defined elsewhere in BSPC
- `WinBSPCPrint` — defined elsewhere; only referenced under `WINBSPC` define
