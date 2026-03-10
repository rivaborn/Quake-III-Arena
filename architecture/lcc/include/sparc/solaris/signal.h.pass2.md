# lcc/include/sparc/solaris/signal.h — Enhanced Analysis

## Architectural Role

This file is part of **LCC's platform abstraction layer** for the SPARC/Solaris architecture. LCC (the C compiler used offline to compile QVM bytecode) requires portable signal handling for its own runtime operation—catching compilation errors, handling segmentation faults, and graceful shutdown. This header bridges LCC's signal-handling code to Solaris's POSIX-like signal API, allowing the compiler toolchain to be self-contained and deployable across multiple Unix platforms without conditional compilation in the main codebase.

## Key Cross-References

### Incoming (who depends on this file)
- Any LCC source files that need signal handling (crash handling, cleanup on SIGINT, etc.)
- Platform-detection mechanism in LCC's build system selects this header for SPARC/Solaris builds
- LCC's standard library initialization code likely initializes signal handlers during startup

### Outgoing (what this file depends on)
- SPARC/Solaris C runtime library (`libc`) — provides the actual `signal()` function and `raise()` implementation
- Solaris kernel — defines the signal numbers and their semantics (SIGABRT, SIGINT, SIGSEGV, etc.)
- No dependencies on other Quake III engine code; this is purely LCC's own infrastructure

## Design Patterns & Rationale

**Platform Abstraction via Directory Structure:** Rather than using `#ifdef SPARC_SOLARIS` guards, LCC uses filesystem-based selection (`lcc/include/sparc/solaris/signal.h` vs. `lcc/include/x86/linux/signal.h`). This keeps the actual compiler code clean and allows the build system to simply append the correct platform directory to the include path.

**Minimal Signal Coverage:** Only 6 signals are defined (SIGABRT, SIGFPE, SIGILL, SIGINT, SIGSEGV, SIGTERM)—the absolutely essential ones for a compiler. This reflects early-1990s POSIX minimalism; modern headers would include SIGUSR1/2, SIGCHLD, SIGALRM, etc.

**Simple Function Pointer Type:** `void (*)(int)` for signal handlers is the classic C89 style—no `sigaction` struct, no flags, no extra context. This is adequate for basic catch-and-exit semantics but offers no signal safety guarantees.

## Data Flow Through This File

1. **Intake:** LCC compiler code (`lcc/src/*.c`) includes this header during compilation
2. **Declaration Only:** The file defines types and function signatures; implementations live in Solaris libc
3. **Output:** Enables LCC to register signal handlers via `signal(SIGINT, handler_fn)` and invoke `raise(SIGTERM)` for controlled shutdown
4. **No State Stored:** This header carries no runtime state; it's purely declarative

## Learning Notes

- **LCC's Self-Containment:** LCC is a **compiler tool**, not part of the Quake III runtime. Its platform headers are entirely separate from the engine's runtime abstractions (`code/win32/`, `code/unix/`, etc.).
- **Era-Specific:** Early 2000s (LCC 4.x era) had minimal signal safety semantics. No `sig_atomic_t volatile`, no `sigprocmask`, no `sigaction`—just raw `signal()`.
- **POSIX Minimalism:** This is compliant POSIX but stripped down. A modern header would expose signal masking, pending signals, and handler context information.
- **Integer Signal Numbers:** Hardcoded signal values (2 = SIGINT, 6 = SIGABRT, 11 = SIGSEGV) are specific to SPARC/Solaris; they differ on x86/Linux (`lcc/include/x86/linux/signal.h` would use different numbers).

## Potential Issues

- **Incomplete `sig_atomic_t`:** Defined as plain `int`, which is **not guaranteed to be atomic** on all SPARC systems. A robust signal handler modifying a `sig_atomic_t` global could still race with the main program on certain architectures.
- **No Signal Masking API:** LCC code cannot safely nest signal handlers or prevent signal delivery during critical sections (no `sigprocmask`). This could cause stack corruption if a signal arrives while LCC is in an unstable state.
- **Limited Coverage:** Omits signals that modern tools expect (SIGUSR1/2 for debugging, SIGCHLD for subprocess management, SIGALRM for timeouts). If LCC ever spawned child processes, handling SIGCHLD would be necessary.
- **Incorrect `signal()` Semantics:** On some POSIX systems, `signal()` resets the handler to SIG_DFL after delivery (BSD 4.2 behavior). Modern code uses `sigaction()` for predictable, repeating handlers.
