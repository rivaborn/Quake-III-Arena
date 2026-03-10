# lcc/include/alpha/osf/string.h — Enhanced Analysis

## Architectural Role

This file is **not part of the runtime engine** but rather part of the **LCC compiler toolchain** (`lcc/` directory). It provides platform-specific C string library declarations for cross-compilation targeting DEC Alpha / OSF/1 systems. During QVM bytecode compilation, LCC uses these headers to validate function signatures and type safety when translating game scripts into architecture-neutral VM instructions.

## Key Cross-References

### Incoming (who depends on this file)
- **LCC preprocessor/compiler** (`lcc/src/`, `lcc/cpp/`) includes this when compiling for Alpha OSF/1 targets
- **Game scripts** (`code/game/`, `code/cgame/`) indirectly depend on correct string function declarations during their compilation to QVM bytecode
- Build system selects this header based on `-target alpha-osf` or similar compiler flag

### Outgoing (what this file depends on)
- **Standard C library** (memory, string manipulation): `memcpy`, `memset`, `strcpy`, `strcat`, etc. — *declarations only; no actual definitions*
- **Compiler's internal size_t type** (guarded against multiple definitions with `_SIZE_T` / `_SIZE_T_`)

## Design Patterns & Rationale

**Multi-platform header abstraction:**
- LCC ships with per-target header directories (`lcc/include/{arch}/{os}/`) to handle ABI and calling-convention differences
- Alpha OSF/1 defines its own `size_t` as `unsigned long` (line 9), which may differ from x86 Linux or other targets
- The `_SIZE_T` / `_SIZE_T_` guard prevents redefinition when multiple headers include `<string.h>`

**Why this structure over modern approaches:**
- Porting era (1990s/early 2000s): cross-compilation required explicit per-platform headers
- Modern practice: use single `#include <string.h>` + conditional compilation (`#ifdef __ALPHA__`) within headers
- This Q3 approach reflects compiler design of that period — each target got its own complete header tree

**Include guard (`__STRING`):**
- Simple single-underscore guard (modern practice: `_STRING_H_`)
- Reflects early C conventions before strict standardization

## Data Flow Through This File

1. **Compile time:** LCC's preprocessor includes this header during QVM script parsing for Alpha OSF/1 target
2. **Type/signature validation:** Compiler verifies that `strcpy(char *, const char *)`, `memcpy(void *, const void *, size_t)`, etc. match usage in game scripts
3. **No runtime effect:** This file contains only *declarations*; no function implementations (those live in the platform's libc at link time, or are not linked into QVM at all)
4. **QVM bytecode generation:** Validated calls are lowered to VM instructions; actual string operations happen in the host engine or platform-specific code

## Learning Notes

**What this teaches about Q3 architecture:**

- **Compiler-centric design:** Unlike modern engines (Unreal, Unity) that precompile scripts offline, Q3's separation of LCC toolchain + QVM runtime reflects late-1990s practices
- **Platform diversity:** The presence of headers for Alpha, MIPS (IRIX), x86 (Linux/Win32), SPARC (Solaris) shows Q3 was engineered for broad hardware support
- **Header-driven cross-compilation:** Rather than runtime feature detection (`#ifdef ALPHA`) or preprocessor unification, each platform gets explicit header files — this is safer for toolchains but scales poorly (evident in the `lcc/include/` tree)
- **Why no `<stdio.h>` in game scripts:** Notice this file only covers memory/string; I/O, process control, and other libc categories are *not* exposed to QVM code — intentional sandbox design
- **Shared `q_shared.h` vs. LCC headers:** Game code uses engine-provided `q_shared.c` / `q_shared.h` (in `code/game/`) for cross-VM utilities, while LCC headers only validate compile-time function signatures

**Modern equivalent:** Today's game engines would use a single unified string header with `#ifdef` guards for ABI differences, or use LLVM/Clang's unified target support rather than per-target header trees.

## Potential Issues

None directly in this file; it is a passive declaration header. However:

- **Unused declarations:** Some functions listed (e.g., `strcoll`, `strerror`) may not be callable from QVM (no evidence they are trapped to the engine). Clarify which are actually exposed.
- **Incomplete libc:** This header provides only string functions; game scripts cannot call printf, malloc, file I/O, etc., suggesting careful API boundary design — verify this is intentional and documented.
