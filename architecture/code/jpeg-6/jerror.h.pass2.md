# code/jpeg-6/jerror.h — Enhanced Analysis

## Architectural Role

This header is part of the vendored IJG libjpeg-6 library, isolated in `code/jpeg-6/` for texture asset loading in the Renderer subsystem. Unlike core engine layers (qcommon, server, client), JPEG is a self-contained tool library that communicates upward only through texture data (via `code/renderer/tr_image.c`). The error infrastructure here enables loose coupling: JPEG never directly invokes engine subsystems; instead, it delegates all error reporting and message emission through a vtable (`jpeg_error_mgr` with `error_exit` and `emit_message` function pointers), allowing the renderer to inject custom behavior at link time.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer texture loader** (`code/renderer/tr_image.c`): only user of JPEG codec; calls functions that trigger these macros internally
- All other `code/jpeg-6/*.c` files: use `ERREXIT*`, `WARNMS*`, `TRACEMS*` macros throughout codec implementation
- No game, server, client, or botlib code directly includes this header

### Outgoing (what this file depends on)
- **`jpeg_error_mgr` vtable** (defined in `code/jpeg-6/jerror.c`): holds `error_exit`, `emit_message` function pointers populated by `jpeg_std_error()`
- **External constants**: `JMSG_STR_PARM_MAX` (from `jpeglib.h`), `JCOPYRIGHT`/`JVERSION` (from `jversion.h`)
- **Standard C**: `strncpy()` for string parameter marshalling
- **Type abstractions**: `j_common_ptr`, `j_compress_ptr`, `j_decompress_ptr` (defined in `jpeglib.h`)

## Design Patterns & Rationale

### Dual-Inclusion Code Generation
The file uses a preprocessor idiom (first pass defines enum, second pass with `JMESSAGE` macro defined builds a string table) that predates C++ templates. This avoids:
- Manual enum-to-string mappings (boilerplate error)
- Two separate data structures drifting out of sync
- Runtime string lookups for all message codes

**Why for JPEG?** JPEG is a standalone library that must ship with tools (q3map, bspc) and the runtime engine. The macro-based generation ensures the toolchain and engine always stay synchronized on message codes, without relying on external build orchestration.

### Vtable Error Dispatch
Rather than calling `exit()` or `fprintf(stderr, ...)` directly, JPEG stores error state (`msg_code`, `msg_parm`) on a context object and invokes a function pointer. This pattern:
- Allows the renderer to inject its own `error_exit` (e.g., `longjmp` to recovery code)
- Lets tools (q3map, bspc) use different error handlers (logging to files)
- Prevents JPEG from forcing application-level semantics

**Contrast with game layer**: The game VM (`code/game`) has no choice of error handler; it must use `trap_Error` syscalls. JPEG, being vendored and tool-reusable, is designed more flexibly.

### Macro Layers by Severity
- **`ERREXIT*`** → fatal, calls `error_exit` (typically `noreturn`), application terminates or `longjmp`s
- **`WARNMS*`** → non-fatal, passes level `-1` to `emit_message`, application logs and continues
- **`TRACEMS*`** → informational, caller picks level (0–9), gated by runtime `jpeg_tracer` levels

This mirrors old-school C library conventions (Apache httpd, OpenSSL) where stderr/syslog is the only output channel.

## Data Flow Through This File

1. **JPEG codec encounters error/warning/trace** → calls `ERREXIT3(cinfo, JERR_BAD_PRECISION, ...)`
2. **Macro writes** to `cinfo->err->msg_code` and `cinfo->err->msg_parm.i[]`
3. **Macro invokes** `(*(cinfo)->err->error_exit)((j_common_ptr)(cinfo))`
4. **`error_exit` function pointer** (registered at codec init by `jpeg_std_error()`) executes
   - If in renderer: typically `longjmp` to a recovery point, freeing partial texture data
   - If in tool (q3map): typically calls `exit(EXIT_FAILURE)` after logging
5. No messages are generated in this header; instead, **`jerror.c` consumes the enum and registers string lookups** in the error manager's message table
6. **`emit_message` callback** looks up `msg_code` in the string table and formats with `msg_parm` values

## Learning Notes

### What a Developer Would Learn
- **C preprocessing as metaprogramming**: The dual-inclusion pattern is a pre-C++ way to generate correlated data structures (enum + strings). Modern C might use designated initializers; C++ would use constexpr or templates.
- **Vendored library decoupling**: JPEG never imports from qcommon, renderer, or game layers. All integration is push-based (the renderer configures JPEG's error manager). This is the cleanest way to embed third-party C code.
- **Era-specific design**: This 1994–1995 code reflects constraints of early 1990s: no C++ standard library, no template metaprogramming, POSIX signal handling via `setjmp/longjmp` as a error recovery primitive. Modern codebases would use exceptions or structured result types.

### Idiomatic to This Engine
- **Clean module boundaries**: Like botlib (also vendored), JPEG is fully self-contained. The renderer imports `<jpeglib.h>` and calls codec entry points; codec calls back through the vtable only.
- **Stateless error codes**: Unlike modern logging frameworks, JPEG error codes are integers with context stored on the codec state object. No thread-local or global state; safe for multithreaded use (each context is independent).

### Game Engine Programming Concepts Not Applied
- **No ECS**: Error handling is state-based (stored in codec structs), not data-driven.
- **No resource handles**: Errors immediately terminate the load; no rollback/deferred cleanup.
- **No error recovery hierarchy**: Either the error handler catches and longjmps, or the application crashes; no error accumulation or reporting chain.

## Potential Issues

1. **`TRACEMS3–8` buffer overflow risk** (minor, internal only):
   - Macros directly index into `cinfo->err->msg_parm.i[]` (8-element int array).
   - If a caller accidentally invokes `TRACEMS9` or higher, memory corruption occurs.
   - Not a practical issue: only codec internals use these, and the library is stable. But it's an example of how macro-based APIs offer no type safety.

2. **Portability of `strncpy` in `ERREXITS`/`TRACEMSS`** (minor):
   - Using `strncpy` (not NUL-terminated by design) to fill a bounded string. Works, but modern safer practices would use `snprintf` or bounds-checked string functions.
   - Not a security issue in JPEG's context (error messages are not attacker-controlled input), but reflects 1990s practices.

3. **No support for dynamic message registration** (design limitation, not a bug):
   - The enum is fixed at compile time; applications can't define custom error codes.
   - This forces all custom errors through generic codes like `JMSG_NOMESSAGE`, then relying on `msg_parm` for context.
   - Acceptable for a closed codec, but would be cumbersome if JPEG were extended with plugins.
