# lcc/include/mips/irix/setjmp.h — Enhanced Analysis

## Architectural Role

This header provides MIPS/IRIX platform-specific declarations for stack unwinding primitives (`setjmp`/`longjmp`). While residing in the LCC compiler tree, the same pattern is critical to **qcommon**'s error handling: the engine's `Com_Error` function uses `longjmp` to unwind the VM call stack and return control to a safe checkpoint, enabling graceful recovery from fatal conditions without propagating exceptions through the entire call chain.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler internals** (transitively, if LCC itself uses setjmp for code generation or runtime support)
- **qcommon/common.c** implicitly depends on the setjmp/longjmp ABI for `Com_Error` error handling (though actual calls are in platform-layer implementations)
- Any other qcommon code that catches `Com_Error` via setjmp/longjmp control flow

### Outgoing (what this file depends on)
- **MIPS/IRIX C library runtime**: provides actual implementations of `setjmp` and `longjmp` (not in this repo)
- No other codebase dependencies; this is a pure platform interface

## Design Patterns & Rationale

**Minimal wrapper approach**: Rather than reimplementing setjmp/longjmp, the header declares the standard C library ABI with platform-specific struct layout (`jmp_buf` as 28-element `int` array). This is the canonical pattern across all `lcc/include/*` platform directories—each target (alpha/osf, mips/irix, sparc/solaris, x86/linux) defines `jmp_buf` to match that platform's register save context size and layout.

**Why 28 ints for MIPS?** Each element likely captures:
- Integer registers (GPRs): ~8–16 registers
- Floating-point registers (if saved): additional slots
- Stack pointer, program counter, control/status regs
- Padding/alignment

This layout must match the actual MIPS/IRIX ABI's `setjmp` implementation exactly, or stack unwinding will corrupt memory or jump to garbage addresses.

## Data Flow Through This File

1. **Compile-time**: LCC parser/code generator includes this header when targeting MIPS/IRIX to know `jmp_buf` layout and function signatures.
2. **Runtime in qcommon**:
   - `setjmp(buf)` called: saves current register state + stack frame into `jmp_buf[28]`
   - `longjmp(buf, val)` called later: restores all registers and jumps back to the `setjmp` site, "returning" `val`
   - Typical use: `if (setjmp(errorStack) == 0) { ... run game frame ... } else { ... handle error ... }`

## Learning Notes

**Portability lessons**:
- Setjmp/longjmp is a low-level, platform-dependent control-flow primitive—not exceptions. Each platform must define `jmp_buf` size to hold all volatile state.
- This reflects late-1990s cross-platform C engineering: no exception-handling language feature; manual try-catch via setjmp macros.
- Modern engines use structured exceptions (C++ try-catch) or error codes; Q3A's use of raw `longjmp` for fatal errors is idiomatic for its era.

**Architectural insight**: The qcommon layer uses `longjmp`-based unwinding to escape deeply nested subsystem calls (VM, collision, parsing) during fatal errors, avoiding cascading cleanup in every caller. This is an alternative to return-code propagation, though less safe (no type-checking of error handlers).

## Potential Issues

- **Stack corruption risk**: If `jmp_buf[28]` is too small for a given MIPS variant (MIPS64, with more FPRs), `setjmp` will overflow, corrupting the next field. This is a silent, hard-to-debug failure mode.
- **ABI mismatch**: If the platform's actual setjmp ABI differs from the declared layout (e.g., ABI changed in a libc update), longjmp will jump to corrupted PC/SP values. The header provides no checksum or version guard.
- **No scope validation**: Code can call `longjmp(buf, ...)` to a `buf` from a different (now-exited) call frame, leaving stale pointers and corrupted local state. Q3A must enforce this discipline via code review.

---

*Note: This file is a thin platform interface; all actual semantics and safety guarantees depend on correct use in qcommon and the MIPS/IRIX C library implementation.*
