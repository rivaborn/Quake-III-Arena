# common/cmdlib.h

## File Purpose
A shared utility header for Quake III Arena's offline tools (q3map, bspc, q3radiant, q3asm). It declares a portable C runtime abstraction layer covering file I/O, string manipulation, path handling, endian conversion, argument parsing, and CRC computation used across all build-time tool executables.

## Core Responsibilities
- Declare cross-platform string utilities (`strupr`, `strlower`, `Q_stricmp`, etc.)
- Declare safe file I/O wrappers (`SafeOpenRead/Write`, `SafeRead/Write`, `LoadFile`, `SaveFile`)
- Declare path manipulation utilities (`ExpandPath`, `ExtractFilePath`, `DefaultExtension`, etc.)
- Provide endian-swap function declarations (`BigShort`, `LittleShort`, `BigLong`, etc.)
- Expose global game/tool directory state (`qdir`, `gamedir`, `writedir`)
- Declare CRC utility functions for data integrity checks
- Provide `qprintf`/`_printf` verbosity-gated output and fatal `Error` reporting

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `qboolean` | typedef enum | Boolean type (`qfalse`/`qtrue`); guarded by `__BYTEBOOL__` |
| `byte` | typedef | Unsigned 8-bit type alias |
| `cblock_t` | struct | Generic 2D data block (pointer + count + width + height) for compression routines |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `myargc` | `int` | global (extern) | Argument count for `CheckParm` |
| `myargv` | `char **` | global (extern) | Argument vector for `CheckParm` |
| `qdir` | `char[1024]` | global (extern) | Base Quake install directory |
| `gamedir` | `char[1024]` | global (extern) | Active game content directory |
| `writedir` | `char[1024]` | global (extern) | Output write directory |
| `com_token` | `char[1024]` | global (extern) | Last token parsed by `COM_Parse` |
| `com_eof` | `qboolean` | global (extern) | EOF flag for `COM_Parse` |
| `archive` | `qboolean` | global (extern) | Whether to archive processed files |
| `archivedir` | `char[1024]` | global (extern) | Archive destination directory |
| `verbose` | `qboolean` | global (extern) | Controls `qprintf` output suppression |

## Key Functions / Methods

### Error
- **Signature:** `void Error( const char *error, ... )`
- **Purpose:** Fatal error reporting; expected to terminate the process.
- **Inputs:** printf-style format string + variadic args.
- **Outputs/Return:** None (no return; terminates).
- **Side effects:** Prints error, exits process.
- **Calls:** Not inferable from this file.
- **Notes:** Tool-side equivalent of engine's `Com_Error(ERR_FATAL)`.

### LoadFile / TryLoadFile / LoadFileBlock
- **Signature:** `int LoadFile(const char *filename, void **bufferptr)` / `int TryLoadFile(...)` / `int LoadFileBlock(...)`
- **Purpose:** Read entire file into a heap-allocated buffer. `TryLoadFile` returns failure gracefully; `LoadFileBlock` aligns allocation to `MEM_BLOCKSIZE`.
- **Inputs:** File path, pointer-to-pointer for output buffer.
- **Outputs/Return:** File byte length; buffer allocated at `*bufferptr`.
- **Side effects:** Heap allocation.
- **Calls:** Not inferable from this file.

### SetQdirFromPath
- **Signature:** `void SetQdirFromPath( const char *path )`
- **Purpose:** Initializes `qdir` and `gamedir` globals by scanning a file path for known game directory markers.
- **Inputs:** A file system path string.
- **Side effects:** Writes `qdir`, `gamedir`.
- **Calls:** Not inferable from this file.

### COM_Parse
- **Signature:** `char *COM_Parse(char *data)`
- **Purpose:** Tokenizer; advances through a text buffer extracting one whitespace/comment-delimited token into `com_token`.
- **Inputs:** Pointer into text data.
- **Outputs/Return:** Pointer past consumed token; sets `com_token` and `com_eof`.
- **Side effects:** Writes globals `com_token`, `com_eof`.

### Endian functions (BigShort, LittleShort, BigLong, LittleLong, BigFloat, LittleFloat)
- Byte-swap primitives used to normalize data between big- and little-endian platforms. Implementations defined elsewhere; all follow the same signature pattern `T Func(T value) → T`.

### Notes
- `qprintf` outputs only when `verbose == qtrue`; `_printf` is unconditional.
- `CheckParm` scans `myargv[0..myargc]` for a named switch string.
- Path utilities (`DefaultExtension`, `StripExtension`, `ExtractFilePath`, etc.) operate in-place or into caller-supplied buffers with assumed `MAX_OS_PATH` capacity.
- `CRC_Init` / `CRC_ProcessByte` / `CRC_Value` implement a streaming CRC-16 over arbitrary byte sequences.

## Control Flow Notes
This is a header-only declaration file for offline tools, not the runtime engine. It participates in tool startup (argument parsing via `myargc`/`myargv`, directory setup via `SetQdirFromPath`) and throughout tool execution (file loading, path resolution, error handling). It has no role in the engine's frame/update/render loop.

## External Dependencies
- **Standard C library:** `<stdio.h>`, `<string.h>`, `<stdlib.h>`, `<errno.h>`, `<ctype.h>`, `<time.h>`, `<stdarg.h>`
- **MSVC-specific:** `#pragma intrinsic(memset, memcpy)`, several `#pragma warning(disable)` suppressions
- All declared functions are **defined elsewhere** (in `common/cmdlib.c` or platform-specific translation units)
