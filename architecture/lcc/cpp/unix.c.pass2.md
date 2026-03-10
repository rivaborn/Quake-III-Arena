# lcc/cpp/unix.c — Enhanced Analysis

## Architectural Role

This file implements the Unix/Linux platform-specific setup and utility layer for the LCC C preprocessor (`lcc/cpp/`), which is part of the offline QVM bytecode compilation toolchain. The `setup()` function bootstraps the preprocessor's initialization phase by parsing command-line arguments, configuring include paths, setting up input/output file descriptors, and initializing the compilation environment. The custom `memmove()` implementation provides a portable memory-movement fallback for systems where vendor libc versions are suboptimal or missing, essential for the self-contained nature of the lcc toolchain.

## Key Cross-References

### Incoming (who depends on this file)
- **`lcc/cpp/cpp.c`** (main preprocessor driver) calls `setup(argc, argv)` during preprocessor initialization
- **lcc linker/build system** invokes the preprocessor binary, triggering `setup()` indirectly through `main()`
- Internal **lcc/cpp/* modules** may depend on global state initialized here (e.g., `includelist`, `Mflag`, `verbose`, `Cplusplus`)

### Outgoing (what this file depends on)
- **`lcc/cpp/cpp.h`** header for preprocessor definitions and macros (e.g., `NINCLUDE`, `Tokenrow`)
- **POSIX libc**: `getopt()`, `open()`, `creat()`, `dup2()`, `strrchr()`, `strlen()` (via `#include <stdio.h>`, `<stdlib.h>`, `<string.h>`, `<stddef.h>`)
- **lcc/cpp/cpp.c internals**: `setsource()`, `unsetsource()`, `maketokenrow()`, `gettokens()`, `doadefine()`, `setobjname()` — all called to set up preprocessor state
- **lcc utility**: `error()` function (FATAL macro) for error handling
- **extern globals** from other lcc/cpp modules: `optarg`, `rcsid`, `includelist[]` array, `verbose`, `Mflag`, `Cplusplus` flags

## Design Patterns & Rationale

**Getopt-based argument parsing**: Follows Unix convention (`getopt()`) for flag processing, not custom parsing. This is idiomatic for 1990s-era C tools. **Flag logic**:
- `-I` prepends to `includelist[]` in reverse order (newer entries override older)
- `-D`/`-U` inject synthetic source lines to define/undefine macros
- `-M` flags dependency generation mode (output include file list)
- `-v`/`-V` provide version and verbose output

**File descriptor setup**: Uses low-level `open()`/`creat()`/`dup2()` rather than `FILE*` abstraction, allowing redirection of preprocessor output without buffering overhead.

**Include path insertion**: `includelist[]` is a fixed-size array (`NINCLUDE`); new `-I` paths are inserted from `NINCLUDE-2` downward, with `includelist[NINCLUDE-1]` reserved as the final entry (likely a default or terminator).

**Custom `memmove()`**: Addresses portability across Unix variants where some systems either lack `memmove()` or implement it inefficiently (e.g., via `malloc`). The implementation handles forward and backward copies to avoid overlapping-region corruption.

**`Cplusplus` flag**: Modular control over C++ mode via `++` flag, enabling per-compilation dialect switching without recompilation.

## Data Flow Through This File

1. **Entry**: `setup()` called from `main()` with `argc`/`argv`
2. **Initialization**: Call `setup_kwtab()` to populate keyword table
3. **Flag processing loop**: `getopt()` iterates over command-line options; each case updates global state:
   - `-N`: marks all "always-include" entries as deleted
   - `-I path`: inserts include path into `includelist[]`
   - `-D name` / `-U name`: creates synthetic source line, tokenizes it, and calls `doadefine()` to register the macro
   - `-M`: enables dependency output; later `setobjname(fp)` records input filename
   - `-v`/`-V`/`+`: increment version/verbose/C++ mode flags
4. **Input file setup**: Default to `stdin` (fd=0); if positional arg given, `open()` it; extract directory path
5. **Output file setup**: Default to `stdout` (fd=1); if second positional arg given, `creat()` it and `dup2()` to fd=1
6. **Final state**: Call `setsource(fp, fd, NULL)` to attach preprocessor to the input file
7. **Return**: Preprocessor proceeds with tokenization and macro expansion

The flow is fundamentally sequential: environment variables, then arguments, then file setup, then preprocessor handoff.

## Learning Notes

**Idiomatic late-1990s Unix C**: This code exemplifies patterns common in pre-POSIX-standardization tools:
- Direct file descriptor manipulation (`dup2`)
- Reliance on `getopt()` for portable argument parsing
- Hand-rolled memory routines (`memmove()`) for portability
- Global flag variables and arrays

**No error recovery**: `open()`/`creat()` failures call `error(FATAL, ...)`, which presumably `longjmp()`s out, terminating compilation immediately. No graceful fallback.

**Offline tool vs. runtime engine**: Unlike the runtime engine (`qcommon/`, `renderer/`), which isolate platform code in `win32/`/`unix/`/`macosx/`, the preprocessor is a single-shot tool; platform abstraction is lighter (just Unix libc).

**Modern comparison**: A modern compiler (e.g., Clang, GCC) would use dynamic arrays or linked lists for include paths, structured option handling, and standard error codes. This is direct, monolithic, but suitable for the Q3 era.

## Potential Issues

None clearly inferable. The code is defensive:
- Bounds check on `includelist[i]` iteration before insertion
- Error on file open/creation failure
- Safe string operations (`strrchr`, length calculation before `newstring`)
- `memmove()` handles zero-length and overlapping regions correctly

The custom `memmove()` returns 0 (not the input pointer), which is non-standard but harmless if the return value is ignored by callers.
