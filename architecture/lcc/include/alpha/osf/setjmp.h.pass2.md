# lcc/include/alpha/osf/setjmp.h — Enhanced Analysis

## Architectural Role

This file is a **platform-specific standard C library header** bundled with the LCC compiler, defining the `setjmp`/`longjmp` interface for the Alpha OSF (DEC Alpha / Tru64 Unix) platform. LCC is the Q3A build-time compiler used exclusively to compile game logic code (game, cgame, ui VMs) to QVM bytecode; this header supports that compilation pipeline. The file is never executed by the runtime engine itself, but rather consumed by the LCC compiler when compiling QVM source code.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC compiler toolchain** during compilation of `code/game/`, `code/cgame/`, `code/ui/`, and their Team Arena counterparts (`code/game_ta.q3asm`, etc.)
- Any QVM source code that includes `<setjmp.h>` directly
- LCC's standard library infrastructure (`lcc/lib/`)

### Outgoing (what this file depends on)
- Declares only standard C functions (`setjmp`, `longjmp`) with no dependencies on other headers
- Defines `jmp_buf` as a platform-specific opaque buffer (`int[84]`)

## Design Patterns & Rationale

**Platform-specific typedef**: The `jmp_buf` size is calibrated to the Alpha OSF ABI (35 integer registers + 1 placeholder + 48 floats = 84 total). This reflects the Alpha architecture's register count and calling convention, which would differ on x86, PPC, or other targets. LCC includes separate headers for each platform variant (`lcc/include/alpha/osf/`, `lcc/include/x86/linux/`, `lcc/include/mips/irix/`, etc.).

**Double-ended compilation**: LCC bundles its own libc headers rather than relying on the system C library. This ensures QVM bytecode is compiled with consistent APIs regardless of the host platform's libc version.

## Data Flow Through This File

No data flows at runtime. At **build time**:
1. QVM source code (e.g., `code/game/ai_chat.c`) may `#include <setjmp.h>`
2. LCC preprocessor resolves to this header based on `-I` search paths during Q3A build
3. LCC parser/codegen compiles `setjmp`/`longjmp` calls into QVM bytecode instructions
4. Runtime engine does NOT link against LCC's libc; instead the runtime qcommon engine provides its own `Com_Error` mechanism (which internally may use host platform's `longjmp` in `code/qcommon/common.c`, but not this header)

## Learning Notes

**Why LCC bundles libc headers**: Early 2000s multi-platform development practice. Rather than assume a system-wide POSIX libc, LCC includes minimal portable headers, ensuring bytecode is generated consistently. The `jmp_buf` typedef is the **critical platform-specific detail**—mismatches would cause stack corruption if `setjmp`/`longjmp` were called with wrong buffer sizes.

**Alpha OSF specificity**: The Alpha ABI (48 FP registers, register windows, specific offsets) is now historical—this codebase targets platforms long obsolete. Modern Quake III ports omit this entirely.

**No runtime engine consumption**: The runtime engine (`code/qcommon/`, `code/client/`, `code/server/`) does NOT use this header. The architecture mentions "longjmp-based Com_Error" in qcommon, but that uses the **host platform's** native `setjmp.h`, not LCC's bundled one. QVM bytecode and the hosting engine are compiled separately with different toolchains.

## Potential Issues

None inferable from code alone. The `jmp_buf` size is a compile-time constant dependent on the Alpha ABI specification. If the value (84 integers) were incorrect, stack corruption would occur only at **runtime if QVM code actually called setjmp/longjmp**—which is rare in Q3A's game logic. Most error handling in the game VM uses exceptions or return codes, not `setjmp`.
