# code/unix/unix_shared.c

## File Purpose
Provides Unix/Linux platform-specific system utility functions shared across the engine — timing, filesystem enumeration, path resolution, and miscellaneous CPU/user queries. It implements the `Sys_*` interface declared in `qcommon.h` for POSIX-compliant platforms.

## Core Responsibilities
- High-resolution millisecond timer via `gettimeofday`
- Sub-frame X11 event timing correction (Linux non-dedicated only)
- Directory creation (`Sys_Mkdir`)
- Recursive and filtered file listing (`Sys_ListFiles`, `Sys_ListFilteredFiles`)
- Platform path resolution: CD path, install path, home path
- Current user and processor count queries
- Optional PPC/Apple `Sys_SnapVector` / `fastftol` fallbacks

## Key Types / Data Structures
None (no new types defined; uses POSIX `struct timeval`, `struct dirent`, `struct stat`, `struct passwd`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cdPath` | `char[MAX_OSPATH]` | static | Stores CD-ROM path set at startup |
| `installPath` | `char[MAX_OSPATH]` | static | Stores installation directory path |
| `homePath` | `char[MAX_OSPATH]` | static | Stores user home/config directory path |
| `sys_timeBase` | `unsigned long` | global | Epoch-relative origin time in seconds for `Sys_Milliseconds` |
| `curtime` | `int` | global | Most recent millisecond timestamp computed by `Sys_Milliseconds` |

## Key Functions / Methods

### Sys_Milliseconds
- **Signature:** `int Sys_Milliseconds(void)`
- **Purpose:** Returns elapsed milliseconds since the first call (sets `sys_timeBase` on first call).
- **Inputs:** None
- **Outputs/Return:** `int` — milliseconds since time origin
- **Side effects:** Writes `sys_timeBase` (once) and `curtime` (every call)
- **Calls:** `gettimeofday`
- **Notes:** Origin wraps ~year 2038 per comment; `curtime` wraps ~24 days.

### Sys_XTimeToSysTime
- **Signature:** `int Sys_XTimeToSysTime(unsigned long xtime)`
- **Purpose:** Converts an X11 event timestamp to engine milliseconds, applying sub-frame correction if `in_subframe` is enabled.
- **Inputs:** `xtime` — X11 `Time` value (ms since Epoch)
- **Outputs/Return:** Engine-relative millisecond timestamp
- **Side effects:** None
- **Calls:** `Sys_Milliseconds`, `Com_Printf` (debug/wrap test path, disabled by `#if 0`)
- **Notes:** Only compiled on `__linux__` non-dedicated. Falls back to `Sys_Milliseconds()` if delta looks anomalous (>30 ms or negative). References external `cvar_t *in_subframe`.

### Sys_ListFilteredFiles
- **Signature:** `void Sys_ListFilteredFiles(const char *basedir, char *subdirs, char *filter, char **list, int *numfiles)`
- **Purpose:** Recursively walks a directory tree, appending entries matching `filter` to `list`.
- **Inputs:** `basedir`, `subdirs` (relative path within base), `filter` (glob-style), `list`/`numfiles` (in-out accumulator)
- **Outputs/Return:** Void; populates `list[]` via `CopyString`, increments `*numfiles`
- **Side effects:** Heap allocation via `CopyString`; max `MAX_FOUND_FILES` (4096) entries
- **Calls:** `opendir`, `readdir`, `stat`, `closedir`, `Com_sprintf`, `Q_stricmp`, `Com_FilterPath`, `CopyString`, self (recursive)
- **Notes:** Skips `.` and `..`; hard cap at `MAX_FOUND_FILES - 1`.

### Sys_ListFiles
- **Signature:** `char **Sys_ListFiles(const char *directory, const char *extension, char *filter, int *numfiles, qboolean wantsubs)`
- **Purpose:** Lists files in a directory optionally filtered by extension, filter glob, or directory-only mode.
- **Inputs:** `directory`, `extension` (or `"/"` for dirs-only), `filter` (delegated to `Sys_ListFilteredFiles` if non-NULL), `numfiles` (out), `wantsubs`
- **Outputs/Return:** Heap-allocated `char **` (NULL-terminated), or NULL if none found
- **Side effects:** Allocates via `Z_Malloc` and `CopyString`
- **Calls:** `Sys_ListFilteredFiles`, `opendir`, `readdir`, `stat`, `closedir`, `Z_Malloc`, `CopyString`, `Q_stricmp`, `Com_sprintf`
- **Notes:** Caller must free with `Sys_FreeFileList`.

### Sys_DefaultHomePath
- **Signature:** `char *Sys_DefaultHomePath(void)`
- **Purpose:** Resolves the user's game data directory (`~/.q3a` on Linux, `~/Library/Application Support/Quake3` on macOS), creating it if absent.
- **Inputs:** None
- **Outputs/Return:** Pointer to static `homePath` buffer, or `""` if `$HOME` is unset
- **Side effects:** May call `mkdir`; calls `Sys_Error` on `mkdir` failure (non-`EEXIST`)
- **Calls:** `getenv`, `Q_strncpyz`, `Q_strcat`, `mkdir`, `Sys_Error`
- **Notes:** Returns cached value if already set.

### Notes (trivial helpers)
- `Sys_Mkdir` — thin wrapper around `mkdir(path, 0777)`
- `strlwr` — in-place tolower loop; asserts on NULL input
- `Sys_Cwd` — `getcwd` into static buffer
- `Sys_SetDefaultCDPath` / `Sys_DefaultCDPath` — getter/setter for `cdPath`
- `Sys_SetDefaultInstallPath` / `Sys_DefaultInstallPath` — getter/setter; falls back to `Sys_Cwd()`
- `Sys_SetDefaultHomePath` — setter for `homePath`
- `Sys_GetProcessorId` — always returns `CPUID_GENERIC`
- `Sys_ShowConsole` — stub (no-op)
- `Sys_GetCurrentUser` — `getpwuid(getuid())`, returns `"player"` on failure
- `Sys_ProcessorCount` (Linux only) — `sysconf(_SC_NPROCESSORS_ONLN)`
- `fastftol` / `Sys_SnapVector` (Apple PPC only) — cast-based truncation and `rint` snap

## Control Flow Notes
This file is a **platform layer** with no frame loop involvement. Functions are called during engine startup (`Sys_DefaultHomePath`, `Sys_DefaultInstallPath`), per-frame event processing (`Sys_Milliseconds`, `Sys_XTimeToSysTime`), and filesystem operations (file listing). It does not participate in render or update loops directly.

## External Dependencies
- **Includes:** `<sys/types.h>`, `<sys/stat.h>`, `<errno.h>`, `<stdio.h>`, `<dirent.h>`, `<unistd.h>`, `<sys/mman.h>`, `<sys/time.h>`, `<pwd.h>`
- **Local headers:** `../game/q_shared.h`, `../qcommon/qcommon.h`
- **Defined elsewhere:** `CopyString`, `Z_Malloc`, `Z_Free`, `Com_sprintf`, `Com_FilterPath`, `Q_stricmp`, `Q_strncpyz`, `Q_strcat`, `Sys_Error`, `Com_Printf`; `cvar_t *in_subframe` (declared `extern`, defined in Linux input code)
