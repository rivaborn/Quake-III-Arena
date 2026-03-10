# code/bspc/l_cmd.c

## File Purpose
A general-purpose command-line and file utility library for the BSPC (BSP Compiler) tool. It provides portable OS abstraction for file I/O, path manipulation, string utilities, argument parsing, byte-order swapping, and CRC computation used throughout the BSP compilation pipeline.

## Core Responsibilities
- Fatal error and warning reporting (console and optional Win32 message box variants)
- File I/O wrappers with error-checked reads/writes and full-file load/save helpers
- Path string manipulation: extraction, extension handling, directory creation
- Command-line argument parsing and wildcard expansion (Win32 only)
- Byte-order (endianness) conversion for short, int, and float primitives
- CCITT CRC-16 computation
- Token parsing from C-string buffers (`COM_Parse`)
- Quake directory resolution from a given path (`SetQdirFromPath`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `qboolean` | typedef enum | Boolean type (`false`/`true`) used throughout |
| `byte` | typedef | Unsigned char alias |
| `cblock_t` | struct | Compression block: pointer to data + byte count (declared in header, not used in this file) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `myargc` | `int` | global | Program argument count, set externally before `CheckParm` |
| `myargv` | `char **` | global | Program argument vector, set externally before `CheckParm` |
| `com_token` | `char[1024]` | global | Last token parsed by `COM_Parse` |
| `com_eof` | `qboolean` | global | Set to true when `COM_Parse` reaches end of input |
| `archive` | `qboolean` | global | When true, `ExpandPathAndArchive` copies files to `archivedir` |
| `archivedir` | `char[1024]` | global | Destination directory for file archiving |
| `qdir` | `char[1024]` | global | Root Quake directory path set by `SetQdirFromPath` |
| `gamedir` | `char[1024]` | global | Game subdirectory path set by `SetQdirFromPath` |
| `verbose` | `qboolean` | global | Controls whether `qprintf` produces output |
| `ex_argc` / `ex_argv` | `int` / `char*[1024]` | global | Expanded wildcard argument storage |
| `crctable` | `unsigned short[256]` | static | Precomputed CCITT CRC-16 lookup table |
| `program_hwnd` | `HWND` | global (WINBSPC only) | Win32 window handle for message box dialogs |

## Key Functions / Methods

### Error
- **Signature:** `void Error(char *error, ...)`
- **Purpose:** Terminates the program after printing a formatted fatal error message.
- **Inputs:** `printf`-style format string and variadic arguments.
- **Outputs/Return:** None (calls `exit(1)`).
- **Side effects:** Writes to log via `Log_Write`, closes log via `Log_Close`, exits process. On WINBSPC: shows a `MessageBox` with `GetLastError()` appended.
- **Calls:** `Log_Write`, `Log_Close`, `exit`, (`MessageBox` on Win32).
- **Notes:** Two platform variants compiled conditionally: WINBSPC (GUI) and console.

### COM_Parse
- **Signature:** `char *COM_Parse(char *data)`
- **Purpose:** Extracts one token from a null-terminated string, skipping whitespace and `//` line comments. Handles quoted strings and single-character delimiters (`{`, `}`, `(`, `)`, `'`, `:`).
- **Inputs:** Pointer to current parse position in string.
- **Outputs/Return:** Pointer advanced past the parsed token; token stored in global `com_token`; sets `com_eof` on end.
- **Side effects:** Writes to global `com_token`; sets `com_eof`.
- **Calls:** None.
- **Notes:** Does not bounds-check `com_token` (fixed 1024 bytes); quoted strings with no closing `"` will loop indefinitely.

### LoadFile
- **Signature:** `int LoadFile(char *filename, void **bufferptr, int offset, int length)`
- **Purpose:** Opens a file, seeks to `offset`, reads `length` bytes (or full file if `length==0`) into a heap allocation, null-terminates it.
- **Inputs:** Path, output pointer-to-pointer, byte offset, byte length.
- **Outputs/Return:** Number of bytes read; `*bufferptr` set to allocated buffer.
- **Side effects:** Allocates memory via `GetMemory`; caller must free.
- **Calls:** `SafeOpenRead`, `fseek`, `Q_filelength`, `GetMemory`, `SafeRead`, `fclose`.

### SetQdirFromPath
- **Signature:** `void SetQdirFromPath(char *path)`
- **Purpose:** Searches the given path string for the literal `"quake2"` directory component to derive and set `qdir` and `gamedir` globals.
- **Inputs:** Absolute or relative file path.
- **Outputs/Return:** None; sets `qdir` and `gamedir` globals.
- **Side effects:** Calls `Error` on failure; calls `qprintf`.
- **Notes:** Hard-coded to search for `BASEDIRNAME = "quake2"` — inconsistent with a Q3 codebase, likely inherited from Q2 tools.

### LittleShort / BigShort / LittleLong / BigLong / LittleFloat / BigFloat
- **Signature:** Various `short/int/float` → `short/int/float` conversions.
- **Purpose:** Byte-order swapping; compile-time selected based on `__BIG_ENDIAN__` / `_SGI_SOURCE` macros. On little-endian hosts the `Little*` variants are pass-through no-ops.
- **Notes:** `SIN` preprocessor guard adds `unsigned short` and `unsigned` variants.

### CRC_Init / CRC_ProcessByte / CRC_Value
- **Purpose:** Stateless CCITT CRC-16 computation using precomputed 256-entry lookup table. Caller maintains the running `unsigned short` CRC state.
- **Notes:** XOR output value is `0x0000`, so `CRC_Value` is effectively a pass-through.

### Notes (minor helpers)
- `Q_strncasecmp`, `Q_strcasecmp`, `Q_stricmp` — portable case-insensitive string comparison.
- `Q_strncpyz` — safe null-terminating `strncpy`.
- `strupr` / `strlower` — in-place ASCII case conversion.
- `CheckParm` — linear scan of `myargv` for a flag string.
- `SafeOpenRead`, `SafeOpenWrite`, `SafeRead`, `SafeWrite` — `fopen`/`fread`/`fwrite` with `Error` on failure.
- `CreatePath` — recursively creates intermediate directories via `Q_mkdir`.
- `QCopyFile` — copies a file by loading it and saving it to a new path.
- `FS_FreeFile`, `FS_ReadFileAndCache`, `FS_FOpenFileRead` — thin shims adapting the Quake filesystem API surface for BSPC.

## Control Flow Notes
This file is a pure utility library with no frame/update/render loop involvement. It is initialized implicitly: callers set `myargc`/`myargv` before `CheckParm`, call `SetQdirFromPath` before any path expansion, and optionally open a log before calling `Error` or `Warning`. It is used throughout the BSPC tool's startup and BSP processing phases.

## External Dependencies
- **Includes:** `l_cmd.h`, `l_log.h`, `l_mem.h`, `<sys/types.h>`, `<sys/stat.h>`, `<direct.h>` (Win32) or `<unistd.h>` (POSIX), `<windows.h>` (WINBSPC), `"io.h"` (Win32 wildcard expansion)
- **Defined elsewhere:** `Log_Write`, `Log_Close`, `Log_Print` (in `l_log.c`); `GetMemory`, `FreeMemory` (in `l_mem.c`); `WinBSPCPrint` (in WINBSPC platform layer)
