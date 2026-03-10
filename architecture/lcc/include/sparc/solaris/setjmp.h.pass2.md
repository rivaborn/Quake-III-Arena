# lcc/include/sparc/solaris/setjmp.h — Enhanced Analysis

## Architectural Role

This platform-specific header provides the C setjmp/longjmp interface for SPARC/Solaris targets, enabling the engine's non-local error recovery mechanism. The `jmp_buf` typedef (12-int buffer) holds register and stack state for the SPARC ABI. Critically, **qcommon's `Com_Error` function relies on `longjmp` to unwind catastrophic failures** back to a safe checkpoint, making this header essential infrastructure for error isolation in a VM-hosting engine that cannot use C++ exceptions.

## Key Cross-References

### Incoming (who depends on this file)
- `code/qcommon/common.c` — implements `Com_Error` using `setjmp`/`longjmp` for error recovery
- Platform-specific compilation flow: included indirectly when building for SPARC/Solaris targets via system header resolution
- Any qcommon-dependent subsystem (renderer, server, game VM) transitively depends on this for crash-safe error unwinding

### Outgoing (what this file depends on)
- C standard library runtime (`libc`) provides actual `setjmp`/`longjmp` implementations
- SPARC/Solaris system ABI defines the layout and semantics of register preservation (why `jmp_buf` is exactly 12 ints)

## Design Patterns & Rationale

**Pattern: Platform Abstraction for Control Flow**  
Rather than inline assembly or compiler builtins, Q3 uses standard C library setjmp/longjmp, maintaining portability across Unix variants. The header tree (`lcc/include/{arch}/{os}/`) mirrors standard system paths, allowing compilation against target platform headers without modifying qcommon source.

**Why this structure:**
- Early 2000s C engine convention: exceptions not available; `longjmp` is the idiomatic "exit to checkpoint" mechanism
- Decouples error recovery from specific platform ABIs; each architecture gets its own jmp_buf size
- Avoids undefined behavior of unwinding stack without proper state capture

## Data Flow Through This File

1. **Initialization phase**: When a qcommon subsystem initializes (e.g., `VM_Alloc`), it calls `setjmp` to capture a safe jump point in a global `jump_buffer`
2. **Normal operation**: Engine runs normally; subsystem calls proceed through stack
3. **Error condition**: Deep in renderer/game logic, `Com_Error(...)` is invoked
4. **Recovery**: `Com_Error` calls `longjmp(jump_buffer, error_code)`, unwinding stack directly to the setjmp site
5. **Cleanup**: Control resumes in qcommon's error handler with stack/registers restored

This pattern isolates VM faults (e.g., interpreter crash, OOM during load) from contaminating the main server/client loop.

## Learning Notes

**Q3-era patterns modern engines abandoned:**
- **Non-local jumps for error recovery** instead of exceptions or return codes. Modern engines prefer Result<T> types or exception hierarchies.
- **Architecture-specific jmp_buf sizing** (12 ints on SPARC, likely different on x86/PPC). Modern C++ uses `std::exception` polymorphism, eliminating ABI coupling.
- **Global jump buffer** (implied) rather than thread-local or context-scoped recovery. This is fine for Q3's single-threaded architecture but fragile for concurrent systems.

**Idiomatic to this engine:**
- The platform header tree (`lcc/include/{arch}/{os}/**/*.h`) is Q3's custom libc; LCC compiler used to produce QVM bytecode, necessitating bundled headers for all target platforms.
- Error recovery is **catastrophic**: once `Com_Error` fires, the entire qcommon module unwinds; fine-grained error handling is not possible.

## Potential Issues

No code-level issues in this header itself. However:

1. **No thread-safety**: A multi-threaded qcommon would have data races on a global `jump_buffer`. Q3 is intentionally single-threaded per server/client instance.
2. **Buffer size coupling**: If an OS/compiler updates its SPARC ABI to require >12 ints for register state, the typedef becomes insufficient. No versioning or runtime check exists.
3. **Silent truncation**: If compiled on a platform where `jmp_buf` is actually larger, the local buffer in qcommon may be overwritten.

These are acceptable tradeoffs for a 2000s shooter engine but indicate why modern engines chose safer error handling schemes.
