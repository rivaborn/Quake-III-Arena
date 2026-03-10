# lcc/include/mips/irix/errno.h — Enhanced Analysis

## Architectural Role
This file is a platform-specific C standard library header for the MIPS/IRIX architecture, part of the LCC compiler's build infrastructure. It declares POSIX error codes and the global `errno` variable used throughout compiled code. The LCC compiler (in `lcc/`) is a *build-time tool* that compiles QVM bytecode for Quake III's virtual machine, completely separate from the runtime engine; this header exists to support that cross-platform compilation toolchain.

## Key Cross-References
### Incoming (dependents)
- Any C code compiled by LCC on IRIX systems would include `<errno.h>` via preprocessor search paths, which would resolve to this file
- Specifically: `lcc/lib/*.c` and any test code in `lcc/tst/` compiled on IRIX
- The LCC compiler itself (`lcc/src/*.c`) may use errno for file I/O or system call error reporting

### Outgoing (dependencies)
- None: this is a pure declaration file with no code dependencies
- The actual `errno` variable is provided by the IRIX C runtime library

## Design Patterns & Rationale
**Per-platform C standard library stubs:** LCC bundles target-platform headers in `lcc/include/{arch}/{os}/`. This is a minimal implementation:
- `EDOM` (33) and `ERANGE` (34) are domain and range errors for math functions
- The `extern int errno;` declaration allows any compiled code to access the thread-unsafe global error state
- No actual errno initialization—that's the C runtime's job

This approach reflects compiler portability circa 2000: LCC needs to compile for multiple architectures (x86, MIPS, SPARC, Alpha), so each gets its own platform-specific include tree. Compare with parallel trees: `lcc/include/x86/linux/`, `lcc/include/sparc/solaris/`, etc.

## Data Flow Through This File
1. **C code compilation phase:** Code written by a developer includes `<errno.h>` (via LCC's `-I` flags pointing to `lcc/include/mips/irix/`)
2. **Preprocessor substitution:** The `#define EDOM 33` and `ERANGE 34` are macro-expanded into the source
3. **Extern resolution:** The `extern int errno` declaration allows the compiled object to reference a global at link time
4. **Runtime binding:** When the resulting binary runs on IRIX, the libc provides the actual errno storage (often thread-local on modern systems, but statically global in this era)

## Learning Notes
- **Era-appropriate:** errno is thread-*unsafe* by design here—no TLS. Modern C uses `__errno_location()` or thread-local storage.
- **Minimal coupling:** LCC doesn't redefine errno values; it copies POSIX/IRIX standard definitions (errno 33=EDOM, 34=ERANGE match IRIX specifications).
- **Compiler infrastructure pattern:** Every hosted compiler (gcc, clang, LCC, etc.) must provide platform-specific C headers. This shows how a modest third-party compiler solved it: one tiny header per platform.
- **Not part of runtime:** Unlike the main Q3 engine (which uses `Com_Error`, `Com_Printf` etc.), LCC doesn't ship with Q3—it's a **build-time dependency only**. The errno declarations here are dead code in a shipped game binary.

## Potential Issues
- **No guards for redefinition:** If code includes both this header and IRIX's system `<errno.h>`, the `extern int errno` declarations will not conflict (both are identical), but macro redefinition of `EDOM`/`ERANGE` could cause surprises if system values differ (they should not, since LCC copies POSIX values).
- **Platform assumption:** The numeric values (33, 34) are correct for IRIX; they differ on other Unix variants (e.g., on Linux, EDOM=33, ERANGE=34 match, but EDOM on BSD may be different). This header is **not portable** to non-IRIX MIPS systems.
