# code/jpeg-6/jinclude.h

## File Purpose
A portability header for the Independent JPEG Group (IJG) JPEG library that centralizes system include file selection and provides cross-platform abstraction macros. It resolves platform differences in standard library availability, string function families, and I/O operations so the rest of the JPEG library can use a uniform interface.

## Core Responsibilities
- Suppresses MSVC compiler warnings when building on Win32 targets
- Conditionally includes system headers (`stddef.h`, `stdlib.h`, `sys/types.h`, `stdio.h`) based on `jconfig.h` feature flags
- Abstracts BSD vs. ANSI/SysV string/memory functions (`bzero`/`bcopy` vs. `memset`/`memcpy`) behind `MEMZERO`/`MEMCOPY` macros
- Provides a `SIZEOF()` macro to guarantee `size_t` return from `sizeof()` on non-conforming compilers
- Provides `JFREAD`/`JFWRITE` macros wrapping `fread`/`fwrite` with portable argument casting

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods
None. This file contains only preprocessor directives, macro definitions, and conditional includes — no functions or variables.

### Notes
- `MEMZERO(target, size)` — expands to `bzero` (BSD) or `memset(..., 0, ...)` (ANSI/SysV) depending on `NEED_BSD_STRINGS`
- `MEMCOPY(dest, src, size)` — expands to `bcopy` (BSD) or `memcpy` (ANSI/SysV)
- `SIZEOF(object)` — casts `sizeof` result to `size_t` for compilers where `sizeof` returns `int`
- `JFREAD(file, buf, n)` / `JFWRITE(file, buf, n)` — argument order is **(file, buf, size)**, reversed from the underlying `fread`/`fwrite` signatures
- The `_WIN32` `#pragma warning` block disables a broad set of MSVC warnings (4018, 4100, 4127, 4244, 4305, 4702, etc.) to allow clean compilation of the IJG C code under the MSVC toolchain

## Control Flow Notes
This is a pure compile-time header. It has no runtime behavior and participates in no engine init/frame/render/shutdown flow. It is included transitively by every other `.c` file in the `jpeg-6` directory via internal JPEG library headers.

## External Dependencies
- `../jpeg-6/jconfig.h` — Watcom-targeted auto-configuration header; defines `HAVE_STDDEF_H`, `HAVE_STDLIB_H`, `NEED_BSD_STRINGS` absence, `NEED_SYS_TYPES_H` absence, and DCT defaults
- `<stddef.h>`, `<stdlib.h>`, `<sys/types.h>` — conditionally included per `jconfig.h` flags
- `<stdio.h>` — unconditionally included (required for `FILE` references in `jpeglib.h`)
- `<string.h>` or `<strings.h>` — selected at compile time for memory/string primitives
- `fread`, `fwrite`, `memset`/`memcpy` or `bzero`/`bcopy` — defined in the C standard library, not in this file
