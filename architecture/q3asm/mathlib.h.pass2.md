Looking at this file in context with the broader codebase, here's the enhanced second-pass analysis:

---

# q3asm/mathlib.h — Enhanced Analysis

## Architectural Role

This header is a **minimal math utility layer for the offline QVM assembler tool**. Unlike the engine's full math infrastructure in `code/qcommon` (which also has physics, collision, and geometric algorithms), the assembler maintains a decoupled, lightweight set of vector and plane primitives. This isolation allows `q3asm` to operate independently from the runtime engine while still having access to basic 3D arithmetic needed for geometric assembly operations.

## Key Cross-References

### Incoming (who depends on this file)
- `q3asm/*.c` — All files in the QVM assembler tool that perform coordinate or plane calculations during bytecode generation
- Historically reused by other tool pipelines (note the parallel math headers in `code/bspc/l_math.h`, `common/mathlib.h`)

### Outgoing (what this file depends on)
- `<math.h>` — Standard C library only; no engine dependencies
- Global `vec3_origin` sentinel (likely defined in some `q3asm` source file)
- No cross-module dependencies; fully self-contained tool support

## Design Patterns & Rationale

**Dual inline/function strategy**: Hot-path operations (`DotProduct`, `VectorAdd`, `VectorSubtract`, `VectorCopy`, `VectorClear`, `VectorScale`, `VectorNegate`) are implemented as **unsafe macros** (no expression-safety, no parameter evaluation order guarantees), while less-frequently-called operations (`VectorLength`, `CrossProduct`, `VectorNormalize`, `PlaneFromPoints`) are function declarations.

This reflects **1990s tool optimization philosophy**: inline macros for maximum speed in tight loops, but functions for complex algorithms. The underscore-prefixed function variants (`_DotProduct`, `_VectorAdd`, etc.) are likely non-inlined reference implementations, possibly used for fallback or validation.

**Platform-agnostic precision control**: The `#ifdef DOUBLEVEC_T` preprocessor switch allows compile-time selection of `float` vs `double` precision, enabling the same code to run as either a fast single-precision tool or a slower high-precision tool depending on build configuration.

## Data Flow Through This File

1. **Type definition layer**: User code includes this header and declares `vec3_t mypos[3]` and `vec4_t myplane[4]` arrays.
2. **Macro expansion**: Inline operations expand at call-site (no function call overhead); compiler optimizes directly into register operations.
3. **Function dispatch**: Complex operations (`VectorNormalize`, `PlaneFromPoints`) resolve to `.o` implementations at link time.
4. **Semantic constants**: `SIDE_*` and `PLANE_*` enums and plane-type classification guide BSP/collision logic in the assembler's geometric passes.

## Learning Notes

**No unsafe macros in modern engines**: This file is a textbook example of 1990s C practice—bare macros with side-effect risk (e.g., `VectorScale(x, i++, out)` evaluates `i++` twice). Modern engines use `inline` functions or C++ templates to eliminate this hazard.

**Plane type classification as a micro-optimization**: The `PLANE_X/Y/Z/NON_AXIAL` constants hint at a common Quake-era trick: axis-aligned planes (e.g., `Z = 5.0`) can be tested much faster than arbitrary planes. The assembler likely queries plane type to optimize trace or collision queries.

**Angle conversion macros**: `DEG2RAD`/`RAD2DEG` suggest the assembler accepts map data in degrees (common in editors) but must convert to radians for internal math operations. Note the slight precision mismatch: `DEG2RAD` uses `180.0F` (float literal) while `RAD2DEG` uses `180.0f`—both correct but inconsistent style.

**Comparison epsilon**: `EQUAL_EPSILON = 0.001` is used throughout the engine to avoid floating-point equality traps in geometric tests. This value (1mm in Quake units) is aggressive—modern engines often use 0.0001 or context-dependent epsilons.

## Potential Issues

**Macro safety**: Macros like `VectorAdd(a, b, c)` can produce incorrect results if arguments have side-effects (e.g., `VectorAdd(v[i++], w, x)`). This is not detectable at compile-time in C89/C99.

**Type aliasing confusion**: `vec2_t` is defined as `vec_t[3]` (3 elements, not 2)—likely a copy-paste error from an earlier version. This may confuse developers expecting a true 2D type.

**Missing bounds checking**: Functions like `AddPointToBounds` and `VectorNormalize` have no guards against zero-length vectors, NaN, or infinity. Division by zero in `VectorNormalize` would silently produce undefined behavior.

**Incomplete const-correctness**: Output parameters (e.g., `void _VectorAdd(..., vec3_t out)`) are not marked `const` in declarations, but callers may assume the input arrays are read-only. No enforcement at the language level.
