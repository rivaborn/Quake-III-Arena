# code/bspc/l_cmd.h

## File Purpose
A utility header for the BSPC (BSP Compiler) tool providing common command-line, file I/O, path manipulation, byte-order conversion, and string utility declarations. It mirrors the pattern of `cmdlib.h` found in other id Software tool codebases.

## Core Responsibilities
- Declare string utility functions (case-insensitive comparison, upper/lower conversion)
- Declare file I/O helpers (safe open/read/write, file loading, path operations)
- Declare byte-order swapping functions (Big/Little endian conversions)
- Declare argument/command-line parsing utilities
- Define shared global state for paths, archive mode, verbosity, and token parsing
- Provide the `qboolean`/`byte` typedefs and a portable `offsetof` macro
- Declare CRC checksum helpers

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `qboolean` | typedef enum | Portable boolean (`false`/`true`) for C89 compatibility |
| `byte` | typedef | Unsigned 8-bit integer alias |
| `cblock_t` | struct | Holds a pointer to byte data and a count; intended for compression routines |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `myargc` | `int` | global (extern) | Argument count for `CheckParm` scanning |
| `myargv` | `char **` | global (extern) | Argument vector for `CheckParm` scanning |
| `qdir` | `char[1024]` | global (extern) | Engine/game root directory path |
| `gamedir` | `char[1024]` | global (extern) | Active game directory path |
| `com_token` | `char[1024]` | global (extern) | Last token parsed by `COM_Parse` |
| `com_eof` | `qboolean` | global (extern) | EOF flag set by `COM_Parse` |
| `archive` | `qboolean` | global (extern) | Enables file archiving mode |
| `archivedir` | `char[1024]` | global (extern) | Destination directory when archiving |
| `verbose` | `qboolean` | global (extern) | Controls verbose `qprintf` output |

## Key Functions / Methods

### Error / Warning
- **`Error(char *error, ...)`** — variadic fatal error; likely calls `exit()`. No return.
- **`Warning(char *warning, ...)`** — variadic non-fatal diagnostic print.

### LoadFile / SaveFile
- **Signature:** `int LoadFile(char *filename, void **bufferptr, int offset, int length)` / `void SaveFile(char *filename, void *buffer, int count)`
- **Purpose:** Allocate and fill a buffer from disk; write a buffer to disk.
- **Inputs:** Filename, pointer-to-pointer for allocation (LoadFile), byte count.
- **Outputs/Return:** LoadFile returns byte count read; SaveFile is void.
- **Side effects:** Heap allocation (LoadFile); file I/O.

### COM_Parse
- **Signature:** `char *COM_Parse(char *data)`
- **Purpose:** Tokenizes a text buffer, writing the current token into `com_token` and setting `com_eof`.
- **Inputs:** Pointer into a text buffer.
- **Outputs/Return:** Pointer advanced past the parsed token.
- **Side effects:** Writes `com_token`, sets `com_eof`.

### Byte-order functions
- `BigShort` / `LittleShort`, `BigLong` / `LittleLong`, `BigFloat` / `LittleFloat` — endian swap helpers; pure functions, no side effects.
- `SIN`-guarded variants add `unsigned short` and `unsigned` overloads.

### Path utilities
- **Notes:** `SetQdirFromPath`, `ExpandArg`, `ExpandPath`, `ExtractFilePath`, `StripExtension`, `DefaultExtension`, etc. are purely string/path manipulation helpers with no notable side effects beyond modifying output buffers.

### CRC helpers
- `CRC_Init`, `CRC_ProcessByte`, `CRC_Value` — incremental CRC-16 computation over byte streams.

## Control Flow Notes
This is a **header-only declaration file**; it participates in init by providing the global path variables (`qdir`, `gamedir`) set early in tool startup via `SetQdirFromPath`. There is no frame/update/render loop involvement — this is a offline build-tool library.

## External Dependencies
- Standard C library: `<stdio.h>`, `<string.h>`, `<stdlib.h>`, `<errno.h>`, `<ctype.h>`, `<time.h>`, `<stdarg.h>`
- All declared functions are **defined elsewhere** (in `l_cmd.c` / `cmdlib.c` within the BSPC tool)
- The `SIN` macro guard enables additional unsigned endian variants originally added for the SiN game engine codebase
