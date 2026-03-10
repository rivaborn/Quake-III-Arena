# lcc/tst/paranoia.c — Enhanced Analysis

## Architectural Role

This file is a **floating-point validation test suite** included in the LCC compiler toolchain, not part of the Q3A runtime engine. It validates IEEE 754 conformance and numerical stability across compilation targets (x86, Alpha, MIPS, Sparc). While not used in the runtime engine, it ensures that the C compiler (`lcc`) produces correct floating-point machine code for all target architectures—critical for the game VM and offline tools that rely on precise math.

## Key Cross-References

### Incoming (who depends on this file)
- **Build system** (`lcc/makefile`, `lcc/buildnt.bat`) — invokes paranoia as a regression test post-compilation
- **Platform-specific makefiles** (`lcc/alpha/osf/tst/`, `lcc/mips/irix/tst/`, `lcc/sparc/solaris/tst/`, `lcc/x86/linux/tst/`, `lcc/x86/win32/tst/`) — test executables stored per-platform with baseline dumps (`.1bk`, `.2bk`, `.sbk` files)
- **Validation workflows** — test output compared against known-good baseline to detect compiler regressions

### Outgoing (what this file depends on)
- **C standard library** (`fabs`, `floor`, `log`, `pow`, `sqrt` via `<math.h>`)
- **POSIX signal layer** (`<signal.h>` for `SIGFPE` trapping; can be disabled with `#define NOSIGNAL`)
- **`setjmp`/`longjmp`** for exception recovery across FPE signals
- **Platform `stdout`** for test output/logging

## Design Patterns & Rationale

### Portable Floating-Point Stress Test
- **Signal-driven exception handling**: wraps FPE in `sigfpe()` handler with `setjmp` recovery to continue testing after hardware exceptions rather than aborting
- **Parameterized precision**: `#define Single` allows compilation as float or double version; macros (`FABS`, `FLOOR`, `LOG`, `POW`, `SQRT`) abstract precision differences
- **Radix/precision auto-detection**: discovers machine radix and unit-in-last-place (ULP) through binary search rather than hardcoding architecture assumptions
- **Systematic error classification**: errors categorized as `Failure` (cardinal arithmetic broken), `Serious`, `Defect`, or `Flaw` for severity escalation
- **Reproducible baselines**: `.1bk`, `.2bk`, `.sbk` suffix pattern suggests before/after/stable baseline dumps for regression detection

### Era-Appropriate Design
- Written in **K&R C** (no function prototypes, minimal type safety) — reflects 1983–1986 era
- **No dynamic memory**: all test state in global FLOAT variables (240+ globals); tests fit small embedded systems
- **Minimal I/O**: only `printf()` and `fflush(stdout)`, no file output (test runner captures stdout)
- **Explicit function stubs**: declares but does not implement `Sign()`, `Random()` — callers must provide

## Data Flow Through This File

**Initialization → Discovery → Stress Testing → Classification**

1. **Initialization** (main entry, lines ~320–360):
   - First assignments use integer RHS to establish baseline constants (0, 1, 2, 3, ...)
   - Recompute float representations to validate compiler constant folding
   - Initialize `ErrCnt[4]` error buckets by severity

2. **Discovery phase** (lines ~380–500+):
   - `Radix` search: increment `W` until `|(W+1)−W|−1| ≥ 1` to find machine base (2, 8, 10, 16, ...)
   - Precision search: multiply `W` by Radix until `(W+1)−W ≠ 1` to find number of digits
   - `U1` (gap below 1.0) and `U2` (gap above 1.0) computation via binary search on relative error

3. **Test phases** (implied by `Milestone` checkpoints and multi-part split structure):
   - Small integer operations (3+3=9, etc.)
   - Radix/precision recalculation via rational-arithmetic sanity checks
   - Underflow/overflow boundary behavior
   - Rounding direction detection (chopped vs. rounded vs. other)
   - Commutative law and associativity stress tests
   - Accumulation error bounds

4. **Output/Classification**:
   - `TstCond()` macro checks conditions; increments `ErrCnt[class]` on failure
   - Prints descriptive messages and diagnostic values (Radix, U1, etc.)
   - Final `ErrCnt` tallies accumulated by severity for summary

## Learning Notes

### What Modern Engines Do Differently
- **Strict IEEE 754 guarantee**: Modern engines assume fixed 64-bit IEEE double and rely on compiler optimizations (e.g., `-ffast-math`)
- **No signal handling**: Instead, NaN/Inf propagates silently; tests use explicit comparisons
- **Platform-agnostic**: Skip machine-discovery phases; assume `std::numeric_limits<T>::epsilon()`
- **Deterministic test runners**: Modern CI integrates unit tests (`gtest`, `pytest`) with coverage reports; paranoia's stdout parsing is manual

### Idiomatic to 1980s Systems Software
- **Portable machine model**: paranoia is instructive precisely because it *discovers* rather than assumes (Radix, precision, rounding mode)
- **FPE signal recovery**: Shows era when hardware FPE exceptions were common; modern x86 rarely traps unless explicitly enabled
- **Manual constant derivation**: Pre-dates symbolic math; deriving `Third = (4/3) − 1` to measure rounding in single operation is clever
- **Milestone checkpoints**: `Milestone = N` globals allow test restart after signal or resource exhaustion (useful on systems with limited memory)

### Game Engine Connection
- Q3A offline tools (`q3map`, `bspc`) use floating-point geometry heavily (plane equations, swept traces, portal adjacency)
- Paranoia running cleanly ensures `lcc` doesn't misbehave for game logic math (weapon ballistics, player physics, collision traces)
- If paranoia fails, Q3A's geometry precision would degrade silently across platforms

## Potential Issues

1. **Global variable side effects**: No restoration of `Radix`, `U1`, `U2` between test runs; if called twice in same process, second run may fail or hang
2. **Signal handler re-entrancy**: `sigfpe()` manipulates global `sigsave`; nested FPE before re-enabling signal may leak context
3. **Single-precision aliasing**: `#define Single` rewrites math functions to cast float↔double; conversions may hide the actual float behavior (e.g., float multiply in a double context)
4. **NOSIGNAL fallback**: Without signal handling, test aborts on FPE; some IEEE violations (e.g., divide-by-zero) might not be detected
5. **Hardcoded thresholds**: `NoTrials = 20` iterations assumes modest accuracy; adversarial rounding sequences might need more trials to trigger

---

**Summary**: Paranoia is a **validation harness for the LCC compiler's floating-point code generation**, ensuring correctness across RISC/x86 architectures. Its sophisticated auto-detection of machine parameters and graceful FPE recovery make it invaluable for 1980s–2000s portability; modern engines assume IEEE 754 compliance by fiat but would benefit from paranoia's methodology for cross-platform graphics/physics engines.
