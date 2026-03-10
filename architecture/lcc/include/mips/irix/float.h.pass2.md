# lcc/include/mips/irix/float.h — Enhanced Analysis

## Architectural Role

This file is a platform-specific standard library header for the **LCC compiler**, a vendored cross-platform C compiler embedded in the Q3 source tree. It defines IEEE 754 floating-point limits for the MIPS IRIX architecture, ensuring deterministic compilation of QVM bytecode regardless of the build host's system headers. Developers writing game code (cgame, game, UI VMs) compiled by LCC can reference these constants to validate precision-critical calculations in bot AI, physics simulation, and networking.

## Key Cross-References

### Incoming (who uses these definitions)
- **Game code files** (`code/game/*.c`, `code/cgame/*.c`, `code/ui/*.c`) compiled by LCC that include `<float.h>` or reference `FLT_MAX`, `DBL_EPSILON`, etc. for range validation
- **Botlib** (`code/botlib/`) physics and reachability calculations that may validate jump arcs or movement predictions against `FLT_EPSILON`
- **q_shared.c** / **q_math.c** utilities that perform floating-point operations and may need to know platform limits

### Outgoing (what this file depends on)
- None directly; this is a pure definition header (no includes, no function calls)

## Design Patterns & Rationale

**Vendored LCC with per-platform headers**: Instead of relying on system `<float.h>` (which could vary across build environments), Q3 includes LCC inline. The `lcc/include/<arch>/<os>/` directory tree mirrors the C standard library structure, providing consistent, reproducible build artifacts across all supported architectures (x86/Linux, MIPS/IRIX, SPARC/Solaris, Alpha/OSF).

**IEEE 754 constants**: The values here (e.g., `FLT_EPSILON = 1.19209289550781250000e-07`, `DBL_MAX_EXP = 1024`) reflect standard IEEE 754 single and double-precision representations.

**Long double aliasing to double**: `LDBL_*` macros map directly to `DBL_*`, indicating that MIPS IRIX treats `long double` as 64-bit (same as `double`). This was common on platforms without distinct hardware support for extended precision.

## Data Flow Through This File

1. **Compile time**: When LCC processes source files, it searches `lcc/include/` for standard headers
2. **Preprocessor substitution**: Code referencing `FLT_MAX` or `DBL_EPSILON` is replaced with literal values before codegen
3. **Runtime**: The resulting QVM bytecode contains hardcoded floating-point limits (no runtime lookups)

## Learning Notes

**Historical context**: MIPS IRIX was used in id Software's development pipeline (SGI workstations); including a vendored compiler meant the studio could guarantee byte-for-byte reproducible builds and deterministic behavior across heterogeneous platforms.

**Reproducibility via vendor control**: By embedding LCC and controlling its standard library headers, Q3 avoided silent differences from system-provided headers. This was critical for networked multiplayer (server/client float precision sync) and offline tools (e.g., bspc must produce identical AAS geometry on any host).

**No modern equivalents**: Today's engines link against system libc or provide their own math libraries; embedding a compiler is rare.

## Potential Issues

None inferable from this file alone. The values are correct IEEE 754 constants. However, code that relies on these macros must ensure:
- Consistent usage across the game and botlib modules (both compiled by LCC)
- No implicit type promotions that change precision between platforms
- Testing on actual MIPS IRIX hardware if determinism across architectures is critical
