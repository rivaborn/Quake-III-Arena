# lcc/include/alpha/osf/stdarg.h — Enhanced Analysis

## Architectural Role

This header is a platform-specific standard library shim for the LCC C compiler targeting Alpha/OSF systems. It is consumed **only** by the LCC compiler toolchain when compiling C source to QVM bytecode—not by the runtime engine itself. The header bridges C language semantics (`va_arg`, `va_start`, `va_end`) to Alpha/OSF's Application Binary Interface (ABI), which passes variadic arguments through a distinct register homing convention incompatible with typical x86/x64 calling conventions.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler frontend** (`lcc/src/expr.c`, `lcc/src/stmt.c`): when parsing/compiling calls to variadic functions, the preprocessor includes this header to resolve `va_*` macros
- **User code compiled by q3asm/LCC**: any Q3VM game/cgame/UI code using `stdarg.h` will indirectly depend on this platform-specific definition
- **Q3A botlib** (`code/botlib/`): bot AI code may use variadic functions internally

### Outgoing (what this file depends on)
- **Compiler builtins**: `__typecode()` macro (compiler-provided intrinsic to query argument types at macro-expansion time)
- No external dependencies; pure macro expansion at compile time

## Design Patterns & Rationale

**Convention-aware macro trampolining**: Rather than abstract the calling convention, this header **embeds Alpha/OSF ABI knowledge directly into macros**. This reflects an era (late 1990s) when:
- Each architecture had radically different calling conventions
- There was no unified psABI (for Alpha, the OSF ABI dominates)
- Compilers expected platform-specific `stdarg.h` per arch/OS tuple

**The ABI model here** (homed arguments + offset tracking):
- First argument is "homed" to a known memory location (`_a0`)
- Remaining arguments are accessed via byte offsets from that base
- Special-case float handling via `_tmp` (floats may be passed in different registers on Alpha, requiring conversion)

**Why the complexity?** Alpha's calling convention passes:
- Integer arguments in `$16`–`$21` (or on stack)
- Floating-point arguments in `$f16`–`$f21`
- Both are "homed" (backed up to memory) per ABI rules

## Data Flow Through This File

1. **Compile time** (LCC expanding user code):
   - User calls `va_start(ap, last_named_arg)` → evaluates `__typecode(__firstarg)` and `__typecode(start)` to determine register vs. stack locations
   - User calls `va_arg(ap, type)` → updates offset, dereferences computed address, handles float↔double conversions via `_tmp`
   - User calls `va_end(ap)` → no-op on Alpha/OSF

2. **Runtime** (Q3VM bytecode execution):
   - The expanded macros are **baked into compiled bytecode**; this header has zero runtime presence
   - No dynamic stack introspection occurs; all offsets pre-computed at compile time

## Learning Notes

- **Architecture-sensitive standard libraries**: This is a museum piece of pre-C99 era. Modern systems use DWARF, frame descriptors, or unwind tables. Alpha/OSF required hardcoding ABI details in every compilation unit.
- **Why LCC carries platform headers**: LCC bundles these because it targets **multiple architectures directly** (see `lcc/include/{alpha,mips,sparc,x86}/*/`). This self-sufficiency meant LCC could bootstrap on any platform without external libc headers.
- **Irrelevance to the runtime engine**: The Quake III engine never interprets `va_*` macros; they're purely a **QVM toolchain artifact**. The engine itself (`code/qcommon/`, `code/server/`, etc.) is compiled with native platform toolchains (MSVC, GCC) and uses their own `stdarg.h` variants.
- **Idiomatic to the era**: The `__typecode()` intrinsic is not standard C but an LCC extension. This shows how compiler-specific extensions were necessary to express ABI quirks before `__VA_ARGS__` and variadic macros became standard (C99).

## Potential Issues

- **Dead code on modern systems**: If Quake III were to be recompiled for Alpha/OSF today, this would work only if LCC itself is still available and functional. No modern project retargets Alpha.
- **Fragility**: The struct definition assumes a specific memory layout; any mismatch between the macro's computed offsets and actual runtime stack layout would cause silent data corruption (e.g., reading floats as integers or vice versa).
- **Float-to-double coercion**: The line `(list)._tmp = (float)*(double *)...` suggests that on Alpha/OSF, some floats are passed as doubles in registers; this narrowing cast could silently lose precision if not carefully matched to actual caller behavior.
