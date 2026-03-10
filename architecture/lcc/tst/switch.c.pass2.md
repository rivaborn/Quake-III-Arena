# lcc/tst/switch.c — Enhanced Analysis

## Architectural Role

This test file validates LCC's switch statement compilation correctness for the entire QVM bytecode pipeline. Since all game logic (server-side `code/game/`, client-side `code/cgame/`, and UI) is compiled to QVM bytecode via LCC, switch statement reliability is a critical correctness invariant. The file exercises edge cases—sparse cases, fall-through patterns, default handling, and integer boundary conditions—that game code relies on for deterministic behavior (e.g., weapon selection FSMs in `code/game/g_weapon.c`, bot decision trees in `code/game/ai_dmnet.c`).

## Key Cross-References

### Incoming (who depends on this file)
- **LCC build system** (`lcc/makefile`, `lcc/buildnt.sh`): Test harness compilation and execution
- **Q3ASM assembler** (`q3asm/`): Validates that LCC's switch bytecode output assembles correctly

### Outgoing (what this file depends on)
- **LCC compiler** (`lcc/src/`, `lcc/cpp/`): Core compilation pipeline (parsing, code generation)
- **C standard library** (implicit): `printf()` for output validation
- **LCC limits.h** (via `#include <limits.h>`): For `INT_MIN`/`INT_MAX` boundary testing

## Design Patterns & Rationale

**Coverage-driven test design**: Each function (`backslash`, `f`, `g`, `h`, `big`, `limit`) targets a specific switch pattern category:
- **`backslash()`**: Compact exhaustive cases (no default, sentinel loop termination)
- **`f()`**: Sparse cases with explicit break statements (most common game code pattern)
- **`g()`**: Multi-label fall-through with default mid-clause (uncommon but tests control flow)
- **`h()`**: Dense loop with many cases but mostly default-handled (stress-tests jump table generation)
- **`big()`**: Unsigned arithmetic and negative case values (edge case in LCC's case-label matching)
- **`limit()`**: Extreme integer values (tests sign-extension and range handling in bytecode)

Why this structure? Early C compilers (including LCC) used simple jump tables for dense cases but linear search for sparse ones. Mixing patterns validates both paths.

## Data Flow Through This File

```
Input:  Compilation by LCC → QVM bytecode → Q3ASM assembly
          ↓
        Test execution: loop/switch iterations
          ↓
Output:  Printf statements to stdout (human-readable correctness verification)
```

The file tests **data invariance through control flow**:
- Variables (`x`, `n`, etc.) are modified conditionally via switch cases
- Output (`printf`) depends on both case matching and fall-through behavior
- Boundary tests (`INT_MIN`, `INT_MAX`) validate sign-bit handling in bytecode comparison operators

## Learning Notes

**For developers studying LCC's QVM pipeline**:
- Switch statement compilation is non-trivial: must distinguish **jump-table** (dense, O(1)) from **linear-search** (sparse, O(n)) strategies
- Q3A's game AI and weapon systems rely on deeply nested `switch` within loops—performance and correctness are coupled
- The `big()` function reveals a subtle LCC/Q3A bug: signed comparison of unsigned bitmask values against negative case labels (line 103–106). This was likely discovered during testing.
- Fall-through patterns (`g()`) demonstrate LCC's lack of warnings for implicit fall-through (modern compilers emit `-Wfall-through`).

**Idiomatic patterns vs. modern engines**:
- Modern engines use dispatch tables or state machines instead of switch chains. Early Q3A used switch liberally (e.g., `ai_dmnet.c`'s FSM).
- This test predates exhaustiveness checking and static analysis of case coverage.

## Potential Issues

1. **Undefined behavior in `big()`** (lines 103–106): Unsigned values are matched against negative literal cases (`-1`, `-2`). In strict C semantics, negative literals have no match in an unsigned switch expression; this likely relies on LCC's implicit sign-extension of case labels. Modern compilers reject or warn.

2. **Fall-through in `g()`** (line 67–69): Cases `6, 7, 8` fall through to `default`. The default clause prints and breaks, but a case *after* default (`case 1001`) is unreachable. Tests LCC's permissiveness with non-standard control flow.

3. **Loop invariant in `h()`** (line 76): The `continue` statement in the default case skips the trailing `printf` for most iterations; only matched cases print. Tests LCC's understanding of `continue` within switch-in-loop.
