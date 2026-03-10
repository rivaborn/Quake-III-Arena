# lcc/include/mips/irix/assert.h — Enhanced Analysis

## Architectural Role

This file is a platform-specific C standard library header for the LCC compiler's MIPS/IRIX target environment. While the engine architecture uses QVM bytecode for portable game logic, LCC must provide a complete C runtime environment to the tools that compile that bytecode. This assert header is part of LCC's self-contained libc replacement — the compiler can target multiple platforms (MIPS/IRIX, x86/Linux, SPARC/Solaris, x86/Win32) and each needs appropriate standard headers.

## Key Cross-References

### Incoming (dependencies on this file)
- **LCC compiler compilation process**: When LCC compiles game VM source (QVM), it includes this header if the target is MIPS/IRIX
- **Game VM code**: Source files in `code/game/`, `code/cgame/`, `code/ui/`, `code/q3_ui/` may call `assert()` during development; LCC resolves these through this header during QVM bytecode compilation
- **LCC toolchain initialization**: `lcc/etc/` platform definition files (`mips.md`, etc.) and `lcc/src/main.c` select the appropriate include directory based on target architecture

### Outgoing (what this file depends on)
- **LCC runtime library**: The `_assert()` function declaration requires a corresponding implementation in LCC's runtime library (typically in `lcc/lib/` or provided at link time)
- **Platform ABI assumptions**: References `__FILE__`, `__LINE__` — compiler built-ins that must be defined; assumes standard calling conventions for `_assert`

## Design Patterns & Rationale

**Standard assert.h dual-mode pattern:**
- **Debug mode** (`#undef` then redefine): Uses `_assert()` runtime function to provide file/line/expression context
- **Release mode** (`NDEBUG` branch): Macro expands to `((void)0)` — completely optimized away, zero runtime cost

**Defensive redefinition:**
The file contains a `#undef assert` before the conditional redefinition. This is intentional: it clears any previous definition (perhaps from a prior include) to ensure the correct mode is active. This prevents double-inclusion conflicts or mode mismatches in complex build scenarios.

**Why a separate implementation per platform?**
LCC predates autoconf and modern header unification. MIPS/IRIX has distinct ABI, calling conventions, and system library behaviors. Duplicating this header across `alpha/osf/`, `mips/irix/`, `sparc/solaris/`, `x86/linux/` allows each platform's assert to match local expectations (e.g., error handling, stack unwinding, signal delivery).

## Data Flow Through This File

**Compile-time (tool invocation):**
1. LCC is invoked with `--target=mips-irix` (conceptually)
2. Include search path selects `lcc/include/mips/irix/` over generic `lcc/include/`
3. Game VM source `#include <assert.h>` resolves to this file
4. If `NDEBUG` is defined (release build): assertion calls compile to no-ops
5. If `NDEBUG` is absent (debug build): `assert(condition)` becomes `((void)((condition)||_assert("condition", file, line)))`
6. QVM bytecode emitted with runtime assertion checks (debug) or removed entirely (release)

**Runtime (QVM execution):**
- In debug mode: Failed assertions call into the engine's trap handler (botlib/game/cgame VMs dispatch assertion failures)
- Release builds produce no runtime overhead

## Learning Notes

**What's idiomatic to this era (early 2000s):**
- **Manual platform-specific includes**: Modern C uses conditional compilation (`#ifdef _IRIX_`) within a single header; LCC uses separate directory trees
- **Redundancy across platforms**: Each platform's assert is nearly identical, but copying was preferred over parameterization
- **Macro-based control flow**: Using `do { ... } while(0)` and inline conditionals rather than inline functions (C89 compatibility, inlining not guaranteed)

**Modern engines differ:**
- Unity/Unreal use unified headers with preprocessor guards: `#if PLATFORM_IRIX`
- C++ assert macros often use variadic macros (`__VA_ARGS__`) for richer context
- Runtime assert filtering/custom handlers are more standard

**Connection to QVM compilation philosophy:**
LCC's platform-specific headers mirror the QVM's design goal: compile once (to portable bytecode), run anywhere (any CPU with QVM interpreter). The LCC headers ensure source compatibility with standard C libraries across compile targets, even though the final QVM output is platform-agnostic.

## Potential Issues

**None clearly inferable**, but worth noting:
- **Symbol collision risk**: If game code or engine code also defines `_assert()`, linker ambiguity could arise during QVM compilation
- **Assertion expression evaluation**: The macro `((e)||_assert(...))` short-circuits on truthy `e`, but only evaluates side effects in `e` once — this is correct C semantics, but complex expressions in assertions can be subtle
- **No sanitizer metadata**: Modern assertion systems (AddressSanitizer, UBSan) can instrument assertions with type info; this header has none
