# common/mathlib.h — Enhanced Analysis

## Architectural Role
This header provides the unified math foundation for all offline tools in the Quake III ecosystem (BSP compiler, AAS compiler, map tools, level editor). It is deliberately **separate** from the runtime engine's `q_shared.h` math library, allowing tools to use optional double-precision computation (`DOUBLEVEC_T`) and tool-specific utilities without affecting the game's real-time constraints. The file serves as the scalar/vector common denominator shared across the `common/` library, `q3map/`, `bspc/`, and `q3radiant/`.

## Key Cross-References

### Incoming (who depends on this file)
- **BSP compiler** (`code/bspc/`): Uses vector types and operations extensively in geometry processing (face merging, area creation, plane classification via `PlaneTypeForNormal`)
- **Map compiler** (`q3map/`): Relies on vector math for brush processing, lighting calculations, and lightmap generation
- **AAS (Area Awareness System)** (`code/botlib/be_aas_*.c`): Depends on plane operations (`PlaneFromPoints`), normal encoding (`NormalToLatLong`), and point rotation for reachability calculation and movement simulation
- **Level editor** (`q3radiant/`): Uses all vector/plane utilities for viewport geometry and entity manipulation
- **Common library** (`common/mathlib.c`, `common/polylib.c`, etc.): Implements and re-exports the declared functions

### Outgoing (what this file depends on)
- **`<math.h>`**: Standard C math library (used by `mathlib.c` implementation, not header itself)
- **`qboolean`, `byte` types**: Defined in a shared baseline header (likely `common/cmdlib.h` or tool-specific typedef)
- **`vec3_origin` global**: Defined in `common/mathlib.c`, provides the canonical zero vector

## Design Patterns & Rationale

### Dual-Precision Strategy
The `DOUBLEVEC_T` conditional allows tools to trade speed for precision when needed (e.g., map compilers doing iterative geometry processing may prefer `double` to avoid accumulated rounding errors). This is **not** available in runtime code, which uses `float` exclusively for memory/performance reasons.

### Macro-Heavy Vector Operations
`DotProduct`, `VectorAdd`, `VectorSubtract`, etc. are **macros, not inline functions**. This reflects 1990s C practice: maximizes inlining without compiler support, avoids function call overhead, and allows clients to avoid the `_` function versions (e.g., `_DotProduct`) which exist only for cases where the macro expands to complex expressions that shouldn't be re-evaluated. The dual API (both macro and function) is idiomatic to this era.

### Plane Representation: `vec4_t`
Planes are stored as `vec4_t` (3 components for normal + 1 for distance). This is space-efficient and aligns with BSP traversal (`PLANE_X/Y/Z` classification for fast axis-aligned tests). The `PlaneTypeForNormal` function is a **fast-path optimization** for BSP algorithms that benefit from knowing whether a plane is axis-aligned.

### Tool-Specific Utilities
Functions like `NormalToLatLong` (2-byte encoded normals for lightmap storage) and `Vec10Copy` (extended vertex data) exist **only** because tools need them; they would never appear in runtime code. This reinforces the separation: `mathlib.h` is a **tool infrastructure** header.

## Data Flow Through This File

1. **Offline compilation phase**:
   - Tools (q3map, bspc) load a BSP and extract geometric primitives (brushes, faces, planes, normals)
   - Vector operations (cross product, normalization, plane construction) are applied to validate geometry and build derived structures (AAS areas, reachability links, lightmap coordinate systems)
   - Normals are encoded via `NormalToLatLong` for compact on-disk storage

2. **Reachability computation (bspc)**:
   - `PlaneFromPoints` constructs planes from area edges
   - `RotatePointAroundVector` simulates bot movement trajectories around pivots (used in jump/ladder validation)
   - `VectorNormalize` ensures consistent normal direction

3. **Runtime usage (NOT via this header)**:
   - The compiled BSP and AAS data are read back by the engine at runtime
   - The engine uses `q_shared.h` math (not this header) to interpret the data

## Learning Notes

### Idiomatic Patterns from the ID Tech 3 Era
- **Macros over functions**: Pre-inlining strategy; assumes compiler optimizations were unreliable
- **Array-of-scalars vectors**: `vec3_t` is `float[3]`, not a struct—optimized for SIMD prefetching on 1999 hardware, though no explicit SIMD code appears in Q3A
- **Single global zero vector**: `vec3_origin` is a constant convenience; modern engines would use `memset` or stack allocation
- **Enum-based plane classification**: `PLANE_X/Y/Z/NON_AXIAL` rather than bit flags; simple and cache-friendly

### Contrast to Modern Engines
- Modern engines use **struct-wrapped vectors** (`struct Vec3 { float x, y, z; }`) for type safety and namespace isolation
- **SSE/SIMD intrinsics** are used directly rather than relying on macros
- **Separate namespaces** (no `_` prefix; use class methods or polymorphism)
- **Quaternions and matrix libraries** are formalized as first-class types (Q3A has minimal matrix support)

### Connections to Engine Concepts
- **BSP spatial partitioning**: Relies on plane math for recursive tree construction and PVS visibility
- **Reachability networks**: Built from plane intersection geometry; forms the waypoint graph for bot pathfinding
- **Lightmap coordinate systems**: UV unwrapping via normal encoding (`NormalToLatLong`) and projection
- **Collision geometry**: `PlaneTypeForNormal` fast-path informs CM (collision model) trace algorithms at runtime

## Potential Issues

### Type Mismatch in `vec2_t` Definition
```c
typedef vec_t vec2_t[3];  // Should be [2], not [3]
```
This is a **latent bug**: `vec2_t` is declared as a 3-element array, contradicting its name and likely intended use. If any code tries to use `vec2_t` for true 2D data (e.g., texture coordinates), it will either:
- Waste a float per vector, or
- Overflow if the third element is accessed

**Impact**: Low in practice (code likely doesn't use `vec2_t` much, preferring raw arrays or `vec3_t`), but it's a API inconsistency that could mislead future maintainers.

### Missing Epsilon Constant Usage
`EQUAL_EPSILON` (0.001) is declared but **no macro or function appears to use it** in the header. The `VectorCompare` function is declared but not inlined; its epsilon tolerance is determined by its implementation in `mathlib.c`, not visible here. This is a documentation/clarity gap rather than a functional bug.

### No Bounds Checking
Macros like `VectorAdd` assume 3-element arrays; there's **no runtime validation** that the caller is passing valid pointers. This is normal for 1990s C, but it's worth noting for safety-critical code (e.g., the BSP compiler might crash on malformed maps rather than gracefully reporting errors).
