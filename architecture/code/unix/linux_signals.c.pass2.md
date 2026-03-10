# code/unix/linux_signals.c — Enhanced Analysis

## Architectural Role

This file is part of the **Unix platform abstraction layer** (`code/unix/`), not the engine core. It handles asynchronous POSIX signals at the OS boundary, converting fatal/termination signals into a controlled engine shutdown sequence. It sits entirely outside the normal game loop and is invoked only on exceptional events, making it a safety mechanism rather than part of the critical path. The file bridges OS-level signal semantics to the engine's `Sys_Exit` abstraction, ensuring clean renderer teardown before process termination.

## Key Cross-References

### Incoming
- **`InitSig()`** — called once during Unix platform initialization, likely from `unix_main.c:Sys_Main()` or similar startup code (not visible in provided cross-reference)
- No other files in the engine directly call the signal handler; it's invoked asynchronously by the OS kernel

### Outgoing
- **`Sys_Exit(int)`** — forward-declared; defined in `unix_main.c` (platform layer abstraction for process termination)
- **`GLimp_Shutdown()`** — from `code/renderer/` (`unix/linux_glimp.c`); conditionally called only in client builds (non-`DEDICATED`)
- **`signal()`** — POSIX C library function (not part of engine)
- **`printf()`** — C stdlib; unsafe in signal handlers but used here for diagnostics

## Design Patterns & Rationale

**Re-entrancy Guard via Static Flag:** The `signalcaught` flag prevents stack overflow if a signal arrives during shutdown. This is a simple, effective 1990s pattern — modern code would use atomic operations or `volatile sig_atomic_t`. The double-fault exit code `1` vs. first-signal code `0` suggests intentional signaling of shutdown mode to parent processes.

**Unified Handler:** One handler for nine signals (SIGHUP, SIGQUIT, SIGILL, SIGTRAP, SIGIOT, SIGBUS, SIGFPE, SIGSEGV, SIGTERM) follows the era's philosophy: minimal complexity, direct action. Modern engines might defer work to a signal-safe queue or use `sigaction()` with proper masking.

**Absence of SIGINT:** Ctrl-C (SIGINT) is conspicuously not registered. This allows terminal-level interrupt to bypass the engine entirely — intentional or oversight is unclear, but it forces users to use kill(1) or Ctrl-C twice (first is ignored, second triggers SIGTERM).

**Conditional Renderer Shutdown:** The `#ifndef DEDICATED` guard reflects Q3's split architecture: dedicated servers have no renderer and skip `GLimp_Shutdown`. This is safe because the renderer DLL is either fully loaded (client) or not loaded (dedicated), with no partial states.

## Data Flow Through This File

1. **Input:** Asynchronous POSIX signal from OS (unpredictable timing, interrupts normal execution)
2. **Filter:** Check `signalcaught` flag:
   - If `false` (first signal): print diagnostic, shut down renderer (client only), call `Sys_Exit(0)`
   - If `true` (second signal): print double-fault warning, force `Sys_Exit(1)` immediately
3. **Output:** Process termination via `Sys_Exit`, bypassing normal frame loop and VM cleanup

## Learning Notes

**Idiomatic to this era:** This is classic late-1990s game engine design — minimal signal handling, trust the OS to terminate on exit, no graceful VM teardown. Modern engines often implement full shutdown sequences (VM cleanup, file flush, memory reporting) in signal handlers using signal-safe operations.

**Async-Signal-Safety Issue:** `printf()` and `GLimp_Shutdown()` are not guaranteed async-signal-safe. A modern port might use `write(STDOUT_FILENO, ...)` instead and defer renderer cleanup to `Sys_Exit()`.

**Contrast with modern patterns:** Today's engines might use `sigaction()` with `SA_SIGINFO`, proper signal masks, and deferred work queues. Q3 opts for immediate, synchronous action — simpler but less robust under cascading failures.

**Connection to platform abstraction:** This file exemplifies how platform layers encapsulate OS-specific concerns. The engine core never mentions signals; the Unix layer handles it. Equivalent `win32/` code would use Windows exception handlers or `atexit()` instead.

## Potential Issues

- **Async-Signal-Safety:** `printf()` and `GLimp_Shutdown()` are not guaranteed safe in signal handlers per POSIX; could corrupt state or deadlock if renderer is mid-operation.
- **Missing `volatile sig_atomic_t`:** The `signalcaught` flag should be `volatile sig_atomic_t` for strict C standard compliance (though modern platforms handle this).
- **Renderer Shutdown Race:** If a signal arrives during `GLimp_Shutdown()` itself, the second signal will force exit without cleanup, potentially leaving the GPU context in an invalid state.
- **No parent notification:** Exit codes 0 vs. 1 attempt to signal mode but don't distinguish the original signal (lost to parent process).
