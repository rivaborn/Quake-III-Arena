# lcc/include/alpha/osf/time.h — Enhanced Analysis

## Architectural Role

This file is a **target-specific C standard library header** for the Alpha/OSF (Digital UNIX) architecture, part of LCC's bundled libc support. LCC is the C compiler used offline to compile QVM (Quake Virtual Machine) bytecode modules (cgame, game, ui). By providing platform-aware type definitions and function declarations, this header allows QVM code to be compiled with standard-compliant time APIs, even though those calls ultimately trap to the engine at runtime via `trap_*` syscalls rather than directly invoking OS time functions.

## Key Cross-References

### Incoming (who depends on this file)
- Any **QVM module source** compiled by LCC (game VM, cgame VM, ui VM) that includes `<time.h>`
- These QVM modules never directly invoke OS time functions; they syscall back to the engine for time queries

### Outgoing (what this file depends on)
- Platform-agnostic C type conventions (`long`, `unsigned long`, `int`)
- No runtime engine dependencies; purely compile-time type/function declarations
- No LCC compiler internals referenced

## Design Patterns & Rationale

**Target-specific guard pattern**: Multiple `#ifndef`/`#define` pairs guard each typedef (e.g., `_CLOCK_T` vs `_CLOCK_T_`) to safely redeclare types across compilation units and handle multiple header inclusions without conflicts — a common practice in 1990s-era libc headers before `#pragma once` standardization.

**Platform-specific extensions**: The `struct tm` includes Digital UNIX–specific fields (`__tm_gmtoff` for UTC offset, `__tm_zone` for timezone name) documented in OSF/1 localtime(3) manpages. These allow higher-fidelity time handling on that platform, though Q3A's game code rarely exercises them.

**Dual-targeted compilation**: This header exists in `lcc/include/{arch}/{os}/` directories (alpha/osf, x86/linux, x86/win32, mips/irix, sparc/solaris). LCC's build system selects the correct variant at compile time, enabling QVM bytecode to be architecture-neutral but compiled with correct type layouts for the *target* CPU on which the engine will run. This decouples QVM semantics from the host build machine.

## Data Flow Through This File

**Compile-time only**:
1. QVM source includes `<time.h>` (via libc chain includes)
2. LCC preprocessor substitutes this header, defining struct layouts and function signatures
3. QVM bytecode compiles with correct stack frame sizes for `struct tm` and correct syscall ABI expectations
4. At runtime, actual `time()` / `mktime()` / `localtime()` calls in QVM trap to the engine's game VM syscall handler, which routes them to native implementations or returns synthesized values

No runtime data actually flows *through* this header — it's pure compile-time shape specification.

## Learning Notes

**LCC's role in Q3A's build pipeline**: Unlike modern engines that use system compilers, Q3A bundled LCC to achieve **reproducible, deterministic QVM bytecode generation**. By controlling the compiler version and libc headers, ID ensured QVM binaries matched across build machines. This header is one of ~50 variant headers enabling that consistency.

**Why `struct tm` matters**: The QVM doesn't directly call localtime(3); instead, game logic may serialize time state or convert timestamps. By defining `struct tm` correctly for the target platform, the bytecode can correctly interpret time data passed from the engine or perform local time calculations.

**Contrast with modern engines**: Today's engines typically compile game code to native DLLs using system toolchains. Q3A's QVM approach (portable bytecode + LCC) was architectural innovation then but imposes this header-management burden.

## Potential Issues

**Struct padding inconsistency**: The `struct tm` definition has a subtle layout issue at line 40:
```c
long __tm_gmtoff;  // <-- No alignment specifier
	char *__tm_zone;   // <-- Tab indentation (possibly indicating error)
```
On some Alpha/OSF configurations, the `long` preceding the pointer may not align correctly if the prior `int` fields don't pack predictably. However, this mirrors actual OSF/1 headers, so it's likely correct as-is; worth verifying against the corresponding system `time.h` from Digital UNIX if bytecode time handling behaves unexpectedly.
