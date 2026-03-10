# code/bspc/l_poly.h — Enhanced Analysis

## Architectural Role

This header is the **geometric kernel** of the BSPC offline compilation pipeline, enabling all CSG, subdivision, face-merging, and area-creation passes. Windings represent convex polygonal faces during BSP→AAS transformation; the function set here is called by `aas_create.c`, `aas_edgemelting.c`, `aas_facemerging.c`, `aas_areamerging.c`, and `aas_gsubdiv.c` to progressively refine the navigation mesh. Unlike the runtime `botlib/be_aas_*.c` reachability and sampling code, `l_poly.h` is **compile-time only** and serves no function in the shipping game engine.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/aas_create.c`**: Builds AAS areas from BSP brushes via winding clipping and validation
- **`code/bspc/aas_edgemelting.c`**: Melts colinear edges and improves face topology via winding operations
- **`code/bspc/aas_facemerging.c`**: Merges coplanar adjacent faces using `TryMergeWinding`
- **`code/bspc/aas_areamerging.c`**: Merges adjacent AAS areas; uses winding queries for adjacency tests
- **`code/bspc/aas_gsubdiv.c`**: Subdivides areas along gravity boundaries and ladder faces; relies on winding clipping
- **`code/bspc/map_q3.c`, `map_q2.c`, etc.**: BSP format parsers that convert raw faces into windings

### Outgoing (what this file depends on)
- Implicit: `mathlib.h` / `q_shared.h` for `vec3_t`, `vec_t`, and math utilities
- **Memory layer**: Custom allocator backing `AllocWinding` (likely in `l_memory.c`)
- **No runtime dependencies**: Windings are compile-time ephemeral; the result is baked into `.aas` binary files

## Design Patterns & Rationale

**1. Flex-Array Allocation**  
The `winding_t` struct declares `vec3_t p[4]` but is allocated with variable size via `AllocWinding(int points)`. This 1990s C technique avoids pointer indirection for vertex lists and keeps geometry cache-friendly. The fixed `p[4]` declaration is a historical artifact; true capacity is determined at allocation time.

**2. Double-Pointer Mutation**  
Functions like `ClipWindingEpsilon(..., winding_t **front, winding_t **back)` and `ChopWindingInPlace(winding_t **w, ...)` accept pointers-to-pointers to allow callers to update their own references. This avoids the caller forgetting to capture the return value, a common C bug pattern. It also signals "the winding may be freed and replaced."

**3. Epsilon-Based Robustness**  
`ON_EPSILON` (0.1 units by default) is a shared tolerance for point-on-plane classification during clipping. This guards against floating-point rounding errors that could create degenerate geometry. The parameter is compile-time configurable for tight geometry.

**4. Explicit Memory Tracking**  
`WindingMemory()`, `ActiveWindings()`, `WindingPeakMemory()` expose heap state for debugging. This was essential in the late 1990s when memory budgets were tight (256–512 MB). Helps detect leaks and profile the compilation pass.

**5. Error Codes Over Exceptions**  
`WindingError()` returns one of 6 `WE_*` codes (e.g., `WE_NONCONVEX`, `WE_DEGENERATEEDGE`); `WindingErrorString()` provides human-readable messages. This reflects the pre-exception era and allows compilation to continue with warnings rather than crashing.

## Data Flow Through This File

```
1. MAP PARSE (map_q3.c)
   ↓ BSP faces → AllocWinding(numverts)

2. CSG / SUBDIVISION (aas_gsubdiv.c)
   ↓ ClipWindingEpsilon() / ChopWindingInPlace()
   ↓ Gravity/ladder boundaries subdivide areas

3. FACE MERGING (aas_facemerging.c, aas_edgemelting.c)
   ↓ TryMergeWinding() attempts coplanar merges
   ↓ RemoveColinearPoints() / RemoveEqualPoints()
   ↓ Topological simplification

4. AREA CREATION (aas_create.c)
   ↓ WindingPlane() extracts plane normals
   ↓ WindingArea() computes face extents
   ↓ CheckWinding() validates no degenerate edges
   ↓ Store computed face data in AAS structure

5. AREA MERGING (aas_areamerging.c)
   ↓ FindPlaneSeperatingWindings() detects adjacency
   ↓ MergeWindings() creates union geometry
   ↓ FreeWinding() releases merged originals

6. CLEANUP
   ↓ ResetWindings() clears global state
   ↓ Verify no leaks via WindingMemory()
```

## Learning Notes

**Idiomatic to this era (late 1990s ID Tech 3):**
- **Manual memory pools** with explicit lifecycle (`AllocWinding`, `FreeWinding`), not malloc/free per-operation
- **Epsilon-based geometric tests** baked into every operation; modern engines now use robust predicates (orient3D, incircle)
- **Explicit error codes** (`WE_*`); modern engines use exceptions or result types
- **Global/module-static state** for memory tracking; modern engines use allocator objects or context parameters
- **Single-pass geometry algorithms**; modern level editors often use spatial acceleration (BSP/KD-trees) for interactive feedback

**Game Engine Concepts:**
- This is a **geometry preprocessing pipeline**, not an ECS or runtime system
- The winding algebra here (clipping, merging, validation) is foundational to **constructive solid geometry (CSG)** and polygon rasterization
- Windings are essentially **planar half-edge structures** without explicit edge/vertex tables—the point list is sufficient for convex polygons
- The `FindPlaneSeperatingWindings()` function encodes a **topological adjacency query**: "are these two faces touching, and along what edge?"

## Potential Issues

1. **Fixed Vertex Limit**  
   `MAX_POINTS_ON_WINDING` (96) is a hard cap. Aggressive clipping subdivision or pathological geometry could exceed this, silently truncating vertices or crashing. Modern engines use dynamic arrays or BSP trees for unbounded complexity.

2. **Epsilon Tuning**  
   `ON_EPSILON` (0.1) is fixed at compile time. Maps with very tight geometry or very large scale may require different values; no per-pass tuning mechanism exists.

3. **No Numerical Predicates**  
   All plane/point classification uses floating-point dot products, susceptible to rounding error accumulation over many clip operations. A single clipped winding chain could compound errors. Robust predicates (e.g., adaptive-precision cross products) would be safer.

4. **Flex-Array Footprint**  
   The `p[4]` declaration is misleading; developers might assume max 4 vertices. Documentation clarity is poor by modern standards.
