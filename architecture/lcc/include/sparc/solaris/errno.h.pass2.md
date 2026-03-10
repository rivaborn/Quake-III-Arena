# lcc/include/sparc/solaris/errno.h — Enhanced Analysis

## Architectural Role

This file is part of the LCC C compiler's **platform-specific standard library header layer** for Solaris/SPARC targets. It provides the error code constants and `errno` global that QVM source code (game, cgame, UI modules) uses when compiled via LCC. The file does **not** execute at runtime; it exists solely to compile QVM bytecode on Solaris systems.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC libc compatibility layer**: Other headers in `lcc/include/sparc/solaris/` (stdio.h, stdlib.h, etc.) may `#include` this during qvm source compilation
- **QVM sources** (code/game, code/cgame, code/ui, code/botlib when compiled on Solaris): Any code that references `errno`, `EDOM`, or `ERANGE`
- **LCC preprocessor/parser**: Processes this during header scanning for Solaris target

### Outgoing (what this file depends on)
- None—this is a leaf header with no dependencies

## Design Patterns & Rationale

**Multi-target platform abstraction**: LCC maintains separate `include/` subdirectories for each supported platform (x86/linux, x86/win32, alpha/osf, mips/irix, sparc/solaris). Each provides the **same interface** (errno.h) with **platform-specific constants**. This allows the same QVM source code to be compiled to bytecode on any platform without modification.

**Minimal surface**: The header defines only two error codes (EDOM, ERANGE) — the subset required by C89/C99 `<math.h>` semantics. This is deliberate: QVM is sandboxed and has no real I/O, so the full POSIX errno set (EACCES, ENOENT, etc.) is unnecessary.

## Data Flow Through This File

```
Compilation:
  QVM source (game/*.c, cgame/*.c, ui/*.c)
    → #include <errno.h>  [resolved to this file on Solaris]
    → LCC preprocessor expands EDOM=33, ERANGE=34
    → QVM bytecode compiled with these constants baked in
    → No runtime role; constants are compile-time tokens
```

The `extern int errno` declaration is cosmetic—QVM has no real errno variable (the VM is sandboxed and never sets it), but the declaration must exist to satisfy C standard conformance and allow code like `errno = 0; math_func();` to compile.

## Learning Notes

**What this teaches about Quake III architecture:**
- The codebase is **extraordinarily portable**: the same QVM source compiles to identical bytecode across Windows, Linux, macOS, and Solaris SPARC via LCC's multi-platform headers
- **Minimal runtime dependencies**: QVM intentionally avoids POSIX-isms; errno is included for C std lib compatibility but rarely used in practice
- **Offline compilation model**: QVM is precompiled to bytecode (during development/distribution), not JIT-compiled at runtime, so header granularity and platform targets matter
- **1990s-era platform awareness**: Supporting Solaris SPARC reflects the era (1999–2005) when SPARC/Solaris was common in studios and servers

This pattern (per-platform headers with shared interface) predates modern approaches like conditional compilation macros (`#ifdef __sparc__`) and is characteristic of the LCC compiler infrastructure from the mid-1990s.

## Potential Issues

**None inferable from code + context**. The file is extremely stable: two hardcoded constants and one standard extern declaration are unlikely to cause problems. The only latent risk is if QVM code *actually* relies on errno semantics (e.g., `perror()` calls), but the sandboxed nature of QVM makes this extremely unlikely.
