# common/polylib.h — Enhanced Analysis

## Architectural Role

`common/polylib.h` is the geometric foundation for Quake III's **offline BSP compilation toolchain**. It defines the convex-polygon (`winding_t`) primitive and all plane-clipping operations that power CSG (constructive solid geometry) subdivision and portal generation in `q3map` and `bspc`. Unlike the runtime engine (client/server/renderer/game), this header is strictly a compile-time tool facility—it has **zero runtime presence** after map compilation completes.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/** (BSP compiler): All brush-to-BSP conversion, CSG operations, portal generation, and lightmap-surface creation rely on winding clipping
- **code/bspc/** (AAS compiler): Uses winding operations during area-face generation from BSP data
- **common/polylib.c** (paired implementation): Contains all function bodies and internal helpers
- No runtime engine subsystems (client, server, game, cgame, renderer) depend on polylib

### Outgoing (what this file depends on)
- **q_math.h / mathlib.h**: `vec3_t`, `vec_t` type definitions and basic vector arithmetic (implicit in plane equations)
- **Makefile/compile flags**: `ON_EPSILON` can be overridden at build time via `-D` flags for different precision requirements per tool

## Design Patterns & Rationale

### Sutherland-Hodgman Clipping (Core Algorithm)
The `ClipWindingEpsilon` and `ChopWinding*` functions implement the **Sutherland-Hodgman convex-polygon clipping algorithm**, which is:
- Deterministic (always produces correct results for convex input)
- Incremental (can be chained: clip by multiple planes sequentially)
- Epsilon-aware (coplanar tolerance prevents numerical instability)

**Why this design**: BSP subdivision requires robust plane splitting. By processing one plane at a time and handling on-plane vertices explicitly, the algorithm avoids epsilon-based decision chains that compound floating-point error.

### Flexible-Array Member Pattern
```c
typedef struct {
    int numpoints;
    vec3_t p[4];    // Declared as size 4 but heap-allocated for variable size
} winding_t;
```
This is a **pre-C99 variable-length array idiom** (predates C99's official `[]` syntax). The struct size is a minimum; callers allocate larger buffers via `AllocWinding(int points)` and cast them as `winding_t*`. This allows single-pointer semantics while avoiding fragmentation from separate vertex arrays.

### Epsilon Tolerance Pattern
`ON_EPSILON` (default 0.1) is a **global coplanarity threshold** used to classify vertices as "on plane" rather than strictly front/back. This:
- Prevents degeneracy (zero-area faces) from floating-point creep
- Is overridable at compile time for different use cases (BSP compiler vs. lightweight collision tools)
- Is **not** a per-operation parameter in the high-level API (callers typically use the compiled-in value)

## Data Flow Through This File

### BSP Compilation Pipeline (Conceptual)
1. **Initialization**: `BaseWindingForPlane(normal, dist)` creates a large initial polygon covering the entire plane
2. **Recursive Subdivision**: For each brush/face:
   - Load/construct windings from BSP entities
   - `ClipWindingEpsilon(...)` clips by splitting plane → front & back fragments
   - Recursively subdivide front/back trees
3. **Portal Generation**: Windings from portal leaves are accumulated and optimized
4. **Cleanup**: `FreeWinding()` at each step to prevent memory bloat during offline compilation

### CSG Boolean Operations
- Brush-to-BSP conversion uses winding clipping to compute intersection/union/difference
- Each clip operation preserves geometric fidelity by respecting the epsilon tolerance

## Learning Notes

### Idiomatic Q3 Engine Patterns
1. **Manual memory management**: `AllocWinding` / `FreeWinding` pairs (no pooling/arena allocation visible at this header level)
2. **Pointer-to-pointer for output parameters**: `ClipWindingEpsilon(..., winding_t **front, winding_t **back)` allows callers to ignore unused results (set to NULL)
3. **In-place operations for efficiency**: `ChopWindingInPlace` avoids stale pointer bugs by taking `winding_t **w` and modifying the caller's variable directly
4. **Plane equation representation**: Implicit throughout—every plane is `(normal, dist)` where `normal · point = dist`

### Historical Context (Pre-Modern Engine Practice)
- No abstraction layer (no `geometry_t` or `polygon_t` base type)
- Direct struct manipulation; no encapsulation
- Epsilon tolerance is compile-time global, not runtime configurable per operation
- Winding orientation (clockwise vs. counterclockwise) is implicit and convention-dependent

## Potential Issues

1. **No runtime bounds enforcement**: `AllocWinding` doesn't validate that `points ≤ MAX_POINTS_ON_WINDING (64)`. Callers must respect this limit or face buffer overflow. The header comment is the only safeguard.

2. **Flexible-array fragility**: The `p[4]` declaration is a lie; the actual size depends on how much memory was allocated. Modern static analyzers may flag this as a potential buffer overrun.

3. **Coplanar epsilon is non-negotiable at runtime**: `ON_EPSILON` is baked at compile time. If a tool needs different tolerance per operation, it must recompile or work around it in higher-level logic.

4. **Silent degeneration in clipping**: If a winding is entirely on one side of a plane, the output pointer (e.g., `*back`) is set to NULL. Callers must always check for NULL results, or crashes occur.
