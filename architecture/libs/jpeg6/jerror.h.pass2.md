# libs/jpeg6/jerror.h — Enhanced Analysis

## Architectural Role

This is a vendored IJG libjpeg-6 header providing the complete error/warning/trace message enumeration and callback-based error reporting macros used throughout the JPEG library. It is consumed *only* by the bundled `code/jpeg-6/` and `code/renderer/tr_image.c` during texture loading—the renderer calls `jload.c` (which uses `jerror.h` macros internally) to decode JPEG frames into GPU textures. As a vendored file, it is architecturally isolated from the engine's native error infrastructure and represents a fixed external dependency rather than a design choice.

## Key Cross-References

### Incoming (who depends on this file)
- `code/renderer/tr_image.c` → indirectly via `jload.c` when decoding JPEG textures during `R_LoadImage`
- All other `code/jpeg-6/*.c` modules use `ERREXIT*`, `WARNMS*`, and `TRACEMS*` macros for error handling
- No direct dependency from core engine (`qcommon`, `server`, `client`) — JPEG handling is renderer-local only

### Outgoing (what this file depends on)
- No dependencies on engine subsystems or globals
- Assumes a `j_common_ptr cinfo` parameter (struct containing `.err` callback vtable) passed by JPEG library caller
- Uses only standard C: `strncpy`, macro composition, no external symbols

## Design Patterns & Rationale

**Multi-pass macro-driven header design:**  
The file uses a clever include-guard+JMESSAGE macro pattern to generate both an enum and string tables from a single source. First include (no JMESSAGE defined) generates `J_MESSAGE_CODE` enum; second include with JMESSAGE redefined generates string tables (as seen in `jcomapi.c`). This pattern, common in late-1990s C libraries, avoids code duplication but is opaque compared to modern approaches.

**Callback-based error model:**  
Unlike the engine's `Com_Error` (which `longjmp`s), JPEG errors invoke `cinfo->err->error_exit` as a function pointer. This decouples JPEG from the engine's exception model—JPEG can be called from multiple contexts (renderer thread, asset tools, offline compilers) without forcing a global error handler.

**Parameterized macro levels:**  
Macros scale from 0 params (`ERREXIT`) to 8 params (`TRACEMS8`) to accommodate formatted messages; `ERREXITS` handles string parameters. This avoids `printf`-style variadic complexity but adds macro bloat—modern engines would use variadic macros or inline functions.

## Data Flow Through This File

1. **Message codes defined here** → embedded in JPEG library calls throughout `code/jpeg-6/` as arguments to `ERREXIT*`, `WARNMS*`, `TRACEMS*`
2. **Error macros expand to:**  
   - Set `cinfo->err->msg_code` (from this enum)
   - Copy optional parameters into `cinfo->err->msg_parm` (fixed-size int/string union)
   - Invoke `cinfo->err->error_exit()` or `emit_message()` callback (provided by caller)
3. **Renderer integration:** `tr_image.c` loads JPEGs via `jload.c`, which internally calls these macros; any error triggers the callback configured by the renderer's JPEG import table

## Learning Notes

**Idiomatic to 1990s C libraries:** This pattern (multi-pass macro, callback-based errors, no exceptions) was the standard for portable C libraries before C99 variadic macros and inline functions. Modern engines (Unity, Unreal, custom) use exceptions, result types (`Result<T, E>`), or explicit error codes—avoiding macros.

**Vendored library isolation:** The file demonstrates how third-party code is "wrapped" architecturally. The renderer doesn't directly call `jerror.h` macros; JPEG internals do. This creates a boundary: errors inside JPEG are handled by its callbacks, not the engine's global error handler. This is both a strength (decoupling) and weakness (harder to unify error logging).

**No modern error handling:** The macros assume a live `cinfo` context is always available and that the error callback is already configured. There is no concept of error codes propagating up the call stack—errors are either fatal (error_exit) or traced (emit_message). This contrasts sharply with the engine's own `trap_*` syscall patterns, which return error codes.

## Potential Issues

- **Vendored static state:** If the JPEG library is ever used from multiple threads without proper synchronization on the `err` callback vtable, races are possible (though unlikely in practice, as each image decode gets its own `cinfo`).
- **No engine integration:** Runtime errors in JPEG decoding are not automatically logged to the engine console or crash reporter—they flow through JPEG's configured error callback, which the renderer must interpret.
- **Macro brittleness:** Changes to message codes or parameter counts require recompilation of all `code/jpeg-6/` modules; there is no versioning or graceful degradation.
