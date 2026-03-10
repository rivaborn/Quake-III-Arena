# common/cmdlib.c

## File Purpose
General-purpose utility library for Quake III's offline tools (q3map, bspc, q3radiant). Provides filesystem I/O, path manipulation, string utilities, argument parsing, byte-order conversion, and CRC computation — the shared foundation for all build/compile-time tools.

## Core Responsibilities
- File I/O: safe open/read/write, load/save whole files, file existence and length queries
- Path manipulation: qdir/gamedir resolution, expansion, stripping, extraction of parts
- String utilities: case-insensitive compare, upper/lower, token parser (`COM_Parse`)
- Command-line argument handling: `CheckParm`, wildcard expansion (Win32 only)
- Byte-order (endian) conversion for short, long, and float
- CCITT CRC-16 computation
- Directory creation and file archiving (`CreatePath`, `QCopyFile`)
- Verbose and broadcast-capable print wrappers (`_printf`, `qprintf`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `qboolean` | typedef enum | Boolean type (`qfalse`/`qtrue`), defined in header |
| `byte` | typedef | `unsigned char` alias |
| `cblock_t` | struct (header-only) | Compression block descriptor; not used in this file |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `myargc` / `myargv` | `int` / `char**` | global | Caller-set program arguments for `CheckParm` |
| `com_token` | `char[1024]` | global | Token buffer filled by `COM_Parse` |
| `com_eof` | `qboolean` | global | Set by `COM_Parse` on end-of-input |
| `archive` / `archivedir` | `qboolean` / `char[1024]` | global | Controls source-file archiving in `ExpandPathAndArchive` |
| `qdir` / `gamedir` / `writedir` | `char[1024]` each | global | Tool working paths set by `SetQdirFromPath` |
| `verbose` | `qboolean` | global | Gates `qprintf` output |
| `ex_argc` / `ex_argv` | `int` / `char*[]` | file-static (Win32) | Expanded wildcard argument storage |
| `crctable` | `unsigned short[256]` | static | Pre-computed CCITT CRC-16 lookup table |
| `hwndOut` / `lookedForServer` / `wm_BroadcastCommand` | Win32 types | global (Win32 only) | IPC handle to "Q3Map Process Server" GUI window |

## Key Functions / Methods

### SetQdirFromPath
- **Signature:** `void SetQdirFromPath(const char *path)`
- **Purpose:** Extracts the engine root (`qdir`) and game subdirectory (`gamedir`) from an arbitrary file path by searching for the `"quake"` base directory name.
- **Inputs:** Any path string (relative or absolute)
- **Outputs/Return:** None; sets globals `qdir`, `gamedir`, `writedir`
- **Side effects:** Writes `qdir`, `gamedir`, `writedir` globals; calls `Error` on failure
- **Calls:** `Q_getwd`, `Q_strncasecmp`, `qprintf`, `Error`, `strcpy`/`strcat`/`strncpy`

### COM_Parse
- **Signature:** `char *COM_Parse(char *data)`
- **Purpose:** Tokenizes a C-like text stream; skips whitespace and `//` comments; handles quoted strings and single special-char tokens (`{}()':`)
- **Inputs:** Pointer into a text buffer
- **Outputs/Return:** Pointer to next position in buffer; token written to `com_token`; sets `com_eof` on end
- **Side effects:** Writes `com_token`, `com_eof` globals
- **Notes:** No buffer-length protection on `com_token` — overflow possible on very long tokens

### LoadFile / LoadFileBlock / TryLoadFile
- **Signature:** `int LoadFile(const char *filename, void **bufferptr)` (and variants)
- **Purpose:** Read entire file into a `malloc`'d buffer. `LoadFileBlock` rounds allocation to 4 KB; `TryLoadFile` returns `-1` instead of erroring on missing files.
- **Inputs:** Filename, pointer-to-pointer for output buffer
- **Outputs/Return:** File length in bytes; `*bufferptr` set to allocated buffer (null-terminated)
- **Side effects:** Heap allocation (caller must `free`)
- **Calls:** `SafeOpenRead`, `Q_filelength`, `malloc`, `SafeRead`, `fclose`

### _printf
- **Signature:** `void _printf(const char *format, ...)`
- **Purpose:** Formatted print that also optionally broadcasts to a Win32 "Q3Map Process Server" GUI via `PostMessage`/`GlobalAddAtom` for progress display in an external monitor window.
- **Side effects:** Writes to stdout; on Win32, may `FindWindow`, `RegisterWindowMessage`, `GlobalAddAtom`, `PostMessage`

### Error
- **Signature:** `void Error(const char *error, ...)`
- **Purpose:** Fatal error handler. Prints message and calls `exit(1)`. On `WIN_ERROR` builds, shows a `MessageBox` with `GetLastError`.
- **Side effects:** Terminates process unconditionally

### Byte-order functions (LittleShort/BigShort/LittleLong/BigLong/LittleFloat/BigFloat)
- Compile-time selected via `__BIG_ENDIAN__` / `_SGI_SOURCE`. On little-endian hosts the `Little*` variants are no-ops; `Big*` perform byte-swaps, and vice versa.

### Notes (minor helpers)
- `copystring` — `malloc` + `strcpy` convenience wrapper
- `Q_strncasecmp` / `Q_stricmp` — portable case-insensitive compare
- `strupr` / `strlower` — in-place case conversion
- `ParseNum` / `ParseHex` — numeric string parsing with hex prefix support (`$`, `0x`)
- `CRC_Init` / `CRC_ProcessByte` / `CRC_Value` — streaming CCITT CRC-16
- `CreatePath` — `mkdir -p` equivalent, Win32 drive-aware
- `QCopyFile` — load + save for file archiving

## Control Flow Notes
This file has no frame/update/render lifecycle. It is a **tool-time library** linked into offline executables (q3map, bspc, q3radiant). Callers invoke `SetQdirFromPath` once at startup to establish working paths, then use the I/O and path utilities throughout processing. `Error` is the single fatal exit point.

## External Dependencies
- `<sys/types.h>`, `<sys/stat.h>`, `<time.h>`, `<errno.h>`, `<stdarg.h>`, `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<ctype.h>`
- Win32: `<windows.h>`, `<direct.h>`, `<io.h>` (for `_findfirst`/`_findnext`, `_getcwd`, `_mkdir`, `FindWindow`, `PostMessage`, `GlobalAddAtom`)
- NeXT: `<libc.h>`
- `cmdlib.h` — declares all exported symbols and defines `qboolean`, `byte`, `MEM_BLOCKSIZE`
- `Q_getwd`, `QCopyFile`, `Q_mkdir` — defined in this file; no external symbols left undefined
