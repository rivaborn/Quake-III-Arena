# code/unix/unix_shared.c — Enhanced Analysis

## Architectural Role

This file is the **Linux/Unix platform abstraction layer** implementing the `Sys_*` interface contract required by the qcommon subsystem. It's one of exactly three compile-time platform implementations (alongside `win32/` and `macosx/`) that provide timing, filesystem, and OS-specific utilities. By isolating platform-specific POSIX calls here, the engine core remains portable; swapping platforms requires only relinking against a different platform layer.

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon/common.c**: Calls `Sys_Milliseconds()` for frame-timing origin and per-frame delta calculations
- **Linux input layer** (likely `unix/linux_common.c` or similar): Defines extern `cvar_t *in_subframe` that `Sys_XTimeToSysTime()` reads
- **Engine startup** (`qcommon/common.c`): Calls `Sys_DefaultHomePath()` and `Sys_DefaultInstallPath()` during initialization to resolve config/install directories
- **Virtual filesystem** (`qcommon/files.c`): Calls `Sys_ListFiles()` and `Sys_ListFilteredFiles()` to enumerate `.pk3` and directory contents
- **Miscellaneous utilities**: `Sys_Cwd()`, `Sys_GetCurrentUser()`, `Sys_Mkdir()`, `strlwr()` called from various qcommon modules

### Outgoing (what this file depends on)
- **POSIX libc**: `gettimeofday()`, `opendir()`/`readdir()`/`closedir()`, `stat()`, `mkdir()`, `getcwd()`, `getpwuid()`, `getenv()`, `sysconf()`
- **qcommon utilities**: `CopyString()`, `Z_Malloc()`, `Z_Free()`, `Com_sprintf()`, `Q_stricmp()`, `Q_strncpyz()`, `Q_strcat()`, `Com_FilterPath()`
- **qcommon error handler**: `Sys_Error()` (fatal exit on critical failures like mkdir permission denial)

## Design Patterns & Rationale

- **Compile-time Platform Abstraction**: The three platform directories (`unix/`, `win32/`, `macosx/`) each implement an identical `Sys_*` interface; only one is linked per build. This avoids runtime polymorphism overhead and keeps implementations simple and platform-native.
- **Static Path Caching**: `cdPath`, `installPath`, and `homePath` are module-statics with lazy setter/getter pairs. Avoids repeated `getenv()` or `getcwd()` system calls and allows callers to override defaults at startup.
- **Lazy Home Directory Materialization**: `Sys_DefaultHomePath()` creates the user config directory (`.q3a` on Linux, `~/Library/Application Support/Quake3` on macOS) only on first call and caches it. Defers I/O until needed.
- **Recursive Glob-Based File Enumeration**: `Sys_ListFilteredFiles()` implements a preorder tree walk with pattern matching. The dual-path design (fast single-directory path via `Sys_ListFiles()` when no glob, recursive path when glob provided) balances common-case speed with flexibility.
- **X11 Sub-Frame Event Timing Correction**: `Sys_XTimeToSysTime()` (Linux non-dedicated only) bridges X11's absolute millisecond timestamps with the engine's relative origin time, applying safety checks for plausible deltas (±30 ms window) to detect anomalies and wrap-around.

## Data Flow Through This File

**Timer flow**: Startup → `Sys_Milliseconds()` (first call only) initializes `sys_timeBase` from `gettimeofday().tv_sec`, returns fractional ms. Subsequent calls compute `(current_sec - sys_timeBase)*1000 + usec_ms`. X11 input events optionally correct timestamps via `Sys_XTimeToSysTime()` if `in_subframe` cvar is enabled, aligning event timing with engine origin.

**Path resolution flow**: Engine startup → calls `Sys_DefaultHomePath()` → reads `$HOME` env var → appends `/.q3a` (or macOS equivalent) → attempts `mkdir()` with cached result. Future calls return cached static buffer. Symmetric setter/getter pattern allows startup code to override defaults before config is loaded.

**File enumeration flow**: Virtual filesystem queries `Sys_ListFiles(dir, ext, glob_filter, ...)` → if glob provided, delegates to recursive `Sys_ListFilteredFiles(basedir, "", glob, ...)` which walks tree, stat-checking each entry and filtering via `Com_FilterPath()`. Otherwise, iterates single directory, appending matching entries to heap-allocated list via `CopyString()`. Returns NULL-terminated array; caller must `Sys_FreeFileList()` to release.

## Learning Notes

- **Classic platform abstraction pattern** (mid-2000s era): This is idiomatic for cross-platform C code before widespread use of build systems that handle per-platform source selection. Modern engines (Unreal, Unity) generalize this via plugin/module loading, but the principle is identical.
- **Lazy path caching** is a foundational systems pattern: avoids repeated expensive syscalls (`getenv`, `getcwd`) by storing results in process-static storage. Safe here because paths don't change at runtime.
- **X11 temporal realignment** (`Sys_XTimeToSysTime`): Solves a subtle problem — X11 reports event times as absolute ms since Epoch; the engine uses session-relative ms. The code detects outliers (delta > 30 ms or negative) as anomalies and falls back to `Sys_Milliseconds()`. Clever because it handles wrap-around gracefully while catching stale/duplicated events.
- **Recursive file enumeration with filtering** is a classic **preorder tree walk**. The `MAX_FOUND_FILES` hard cap (4096) is a safety valve against infinite or pathological directory structures.
- **Q_strncpyz/Q_strcat idioms**: Quake-specific safe string functions predate standard `strlcpy`/`strlcat` and enforce null-termination even on truncation—a defensive coding pattern from the 1990s.

## Potential Issues

- **Year 2038 problem**: `sys_timeBase` is `unsigned long`, storing seconds since Epoch. The comment acknowledges wrapping at `0x7fffffff` (~year 2038). On 64-bit platforms this is deferred indefinitely, but on 32-bit systems this becomes a hard deadline. Requires type change (backwards-compat risk).
- **X11 wrap-around validation incomplete**: The disabled wrap-around test path (`#if 0`) suggests sub-frame timing correction was never fully validated under real wrap-around conditions (49-day X11 Time period).
- **Fatal error on home directory mkdir failure**: If `mkdir(homePath, 0777)` fails with any error except `EEXIST`, the engine calls `Sys_Error()` and exits. Permission/filesystem errors thus crash the game rather than falling back gracefully (e.g., to `/tmp`).
- **Silent path truncation in `Sys_Cwd()`**: The function doesn't check `getcwd()` return value; overflow silently truncates at `MAX_OSPATH-1`, potentially returning an invalid path.
- **Hardcoded fallback for missing user**: `Sys_GetCurrentUser()` returns hardcoded `"player"` on any `getpwuid()` failure without logging the failure reason, obscuring configuration problems.
