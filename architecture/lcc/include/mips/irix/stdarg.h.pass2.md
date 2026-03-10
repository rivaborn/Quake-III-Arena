# lcc/include/mips/irix/stdarg.h — Enhanced Analysis

## Architectural Role

This file is part of the **LCC compiler's platform-specific standard library headers**, not the Quake III runtime engine itself. LCC is the offline toolchain that compiles C source code for the three QVM bytecode modules (cgame, game, ui) which execute within the `qcommon/vm.c` VM host. As a MIPS/IRIX-specific header, it provides variable-argument (`va_list`) support for any QVM code that uses `printf`-style varargs during compilation. The header is selected at build time based on the target architecture and ensures that compiled QVM bytecode correctly implements calling conventions.

## Key Cross-References

### Incoming (dependencies on this file)
- Any C source compiled to QVM bytecode that uses `<stdarg.h>` (e.g., logging, debug output in `code/botlib`, `code/game`, `code/cgame`)
- The LCC compiler's preprocessor (`lcc/cpp/`) includes this when `#include <stdarg.h>` is encountered
- Build system selects this header variant based on `MIPS` + `IRIX` platform flags during QVM compilation

### Outgoing (what this file depends on)
- No external dependencies; this is a pure macro/typedef definition
- Relies on C preprocessor and compiler knowledge of stack layout and calling conventions
- The macros assume the compiler will resolve `sizeof()` and pointer arithmetic at compile time

## Design Patterns & Rationale

**Calling convention encoding in macros**: The `va_start` macro uses inline arithmetic to detect small vs. large argument sizes and adjusts the `va_list` pointer accordingly. This is necessary because MIPS calling conventions pass values smaller than 4 bytes in the lower bytes of 4-byte stack slots, requiring different pointer offsets.

**Endianness split**: The use of separate `_bigendian_va_arg` and `_littleendian_va_arg` reflects IRIX's big-endian byte order. The 64-bit alignment (`(int)list + 15)&~7U`) and special handling for 1-byte and 2-byte types extract values from the correct bytes of aligned stack words—a pattern extinct in modern little-endian systems.

**Minimal implementation**: `va_end` is a no-op (`((void)0)`), which was common in older systems that didn't require cleanup; modern varargs may use it for stack unwinding.

## Data Flow Through This File

1. **Compilation phase**: When `q3asm` or LCC processes game/cgame/botlib source (e.g., `G_Printf`, `Com_Printf`), the C preprocessor includes this header.
2. **Macro expansion**: During preprocessing, varargs macros are expanded inline at every call site:
   - `va_start(ap, fmt)` → pointer arithmetic to locate first variadic argument
   - `va_arg(ap, type)` → pointer advancement and type-safe dereferencing
3. **QVM bytecode**: The expanded code is then compiled to QVM bytecode, embedding the stack manipulation logic into the executable module.
4. **Runtime execution**: When the VM executes, the `va_arg` pointer adjustments occur within the sandboxed VM context (bounded by `dataMask`).

## Learning Notes

**Historical platform specificity**: This header captures 1990s–2000s MIPS/big-endian systems. Modern Quake engines target x86/ARM (little-endian), where argument passing and stack alignment differ significantly. The IRIX header is archaeologically interesting: it shows how varargs were manually implemented before C99/C11 `<stdarg.h>` became ubiquitous.

**Compiler-internal knowledge**: The macros encode assumptions about:
- Stack growth direction (downward on MIPS)
- Word size (4 bytes for `int`, 8 for 64-bit values)
- Alignment (4-byte slots, 8-byte alignment for doubles)
- Endianness (big-endian with special-case handling for small types)

These are not portable; copying them to a different architecture without understanding MIPS ABI would cause silent memory corruption.

**Contrast with modern engines**: Contemporary game engines (Unreal, Unity) either use C++ variadic templates or rely on standard C11/C17 `<stdarg.h>`. The QVM's need for pre-standard varargs reflects its era (Quake III: 1999–2005).

## Potential Issues

- **Unused on non-MIPS platforms**: If someone builds QVM bytecode on x86 or ARM hosts, this header is bypassed. The correct platform-specific header must exist and be selected, or compilation will fail.
- **ABI fragility**: Any change to the MIPS calling convention (alignment, register allocation, etc.) would break these macros silently—they embed hardcoded stack offsets.
- **64-bit pointer truncation**: The cast `(int)list` on line 20 truncates a pointer to `int`, relying on 32-bit MIPS addressing. On 64-bit systems, this causes undefined behavior.
- **Endianness assumption not documented**: Code that includes this and later switches platforms may inherit the big-endian assumptions unintentionally.
