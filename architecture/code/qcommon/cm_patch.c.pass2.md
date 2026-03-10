# code/qcommon/cm_patch.c — Enhanced Analysis

## Architectural Role

This file implements **offline Bezier patch collision preprocessing** for `qcommon` — the runtime integration layer. It receives control-point grids from BSP patches (via `cm_load.c`), performs quadratic curve subdivision until linear approximation is valid, and emits a finalized `patchCollide_t` structure consumed by the runtime trace system. This sits within the broader collision subsystem (`cm_*.c`), which spans BSP loading, swept-volume tracing, and spatial queries; patches are a specialized geometry type requiring curve-to-plane conversion because Quake III's collision model (plane-based, facet-centric) cannot represent Bezier surfaces directly.

## Key Cross-References

### Incoming
- **`cm_load.c` / `CM_LoadMap`**: Calls `CM_GeneratePatchCollide` for each patch surface parsed from BSP; stores result in `cPatch_t` array
- **`cm_trace.c`**: Calls `CM_TraceThroughPatchCollide` and `CM_PositionTestInPatchCollide` during per-frame trace iteration
- **Renderer** (`tr_curve.c`): Loads and renders patch surfaces independently (no collision dependency)

### Outgoing
- **`cm_polylib.h`** (winding utilities): `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `FreeWinding`, `CopyWinding` — used in `CM_AddFacetBevels` to compute edge bevels
- **`qcommon` core**: `Hunk_Alloc` (result allocation), `Com_Error`, `Cvar_Get` (debug cvars), `AddPointToBounds`/`ClearBounds` (AABB computation)
- **Math library**: `VectorSubtract`, `CrossProduct`, `VectorNormalize`, `DotProduct`, `VectorMA`

## Design Patterns & Rationale

**Pattern: Compile-Time Geometry Flattening**  
The core insight is that Bezier curves are *preprocessed offline* into linear-segment approximations and planes, enabling *runtime traces to use only plane-AABB tests*. This avoids expensive curve-intersection math per trace; instead, the subdivision error bound (`SUBDIVIDE_DISTANCE = 16` units) is "paid" once at load time. The tradeoff: higher memory cost (planes × facets) versus CPU cost at trace time.

**Why this design?**  
In 1999–2005, per-frame swept-volume collision against arbitrary curves was prohibitively expensive on CPUs without SIMD or GPU assistance. The Quake III approach (coplanar facet tessellation + plane-based traces) is idiomatic to its era: see also Valve's Source engine, which similarly tessellates curved geometry at compile time.

**Modern contrast:**  
Contemporary engines (Unreal, Unity) use either hierarchical collision BVH trees, GPU-accelerated curve tracing, or continuous implicit-surface solvers — all post-date CPU-limited era constraints.

**Bevel-plane technique:**  
`CM_AddFacetBevels` adds "edge bevels" to prevent sweeping boxes from slipping through cracks between adjacent facets. This is a **crack-prevention pattern** common in game collision; the alternative (tessellation refinement) would increase memory proportionally.

## Data Flow Through This File

```
[BSP patch in memory]
    ↓ (width, height, points[])
[CM_GeneratePatchCollide]
    ├─ CM_SetGridWrapWidth: detect U/V wrap for toroidal patches
    ├─ CM_SubdivideGridColumns: recursively bisect columns if curvature > threshold
    ├─ CM_RemoveDegenerateColumns: collapse identical columns
    ├─ CM_TransposeGrid: swap axes and repeat (subdivide rows)
    ↓ [fully subdivided grid, all edges linear within ε]
[CM_PatchCollideFromGrid]
    ├─ iterate quad cells (i,j) → (i+1,j+1)
    ├─ build triangle plane normals (signbits for AABB offset lookup)
    ├─ assign border planes (surface + axial + edge bevels)
    ├─ validate facets (coplanar points? degenerate normal?)
    ↓ [all facets have surface + bevel planes]
[Hunk_Alloc patchCollide_t, copy planes/facets]
    ↓ [returned to caller, cached in cPatch_t]

[At trace time: cm_trace.c]
CM_TraceThroughPatchCollide(tw, pc):
    for each facet in pc->facets:
        CM_CheckFacetPlane(tw, facet)  ← swept-AABB vs planes
```

**Key invariant:** Every quad cell produces 2 triangles (or 1, if degenerate); each triangle is assigned one `patchPlane_t` (surface) plus N border planes. The border planes form a "wall" around each triangle to catch tunneling.

## Learning Notes

### Idiomatic Patterns to Quake III
- **Magic number thresholds**: `SUBDIVIDE_DISTANCE=16`, `PLANE_TRI_EPSILON=0.1`, `WRAP_POINT_EPSILON=0.1`, `NORMAL_EPSILON=0.0001`, `DIST_EPSILON=0.02` — typical of era-2000 codebases; would be parameterized in modern engines.
- **Signbits for AABB offset**: The `signbits` field in `patchPlane_t` encodes the plane normal's sign bits, enabling fast AABB-side queries without per-axis branching. This is a micro-optimization specific to box-vs-plane tracing.
- **Dual-grid transposition**: Rather than 2D subdivision, the code subdivides columns, transposes, and recurses on rows. This avoids 2D recursion bookkeeping and fits the control-point layout.

### Connections to Engine Concepts
- **Tessellation overhead vs. quality**: The choice of `SUBDIVIDE_DISTANCE` controls tessellation density; larger values = fewer planes but coarser approximation. This is the fundamental tradeoff in any geometry LOD system.
- **Bevel planes vs. edge collapse**: Edge bevels trade memory (more planes) for robustness (no gaps). Alternative: weld vertices below epsilon, which sacrifices precision.
- **Hunk allocation**: All results live on the high-water mark hunk (`h_high`), freed at map shutdown. This is characteristic of Quake's "arena allocator" memory model.

## Potential Issues

1. **Fixed-size working arrays** (`planes[MAX_PATCH_PLANES]`, `facets[MAX_FACETS]`): If a complex patch generates >2048 planes or facets during `CM_PatchCollideFromGrid`, the code calls `Com_Error` and terminates load. No graceful degradation or overflow recovery.

2. **Subdivision epsilon coupling**: The `SUBDIVIDE_DISTANCE` constant is hardcoded; there's no mechanism to adjust tessellation density per-map or per-patch without recompilation. Modern engines expose this as a quality slider.

3. **Degenerate row/column warnings**: The file header comments that behavior may be incorrect for meshes with only a few degenerate triangles in a row/column; the code assumes mostly-regular grids. Edge cases (e.g., a single degenerate column flanked by normal ones) may leave artifacts.

4. **Debug rendering overhead**: The `debugPatchCollide` / `debugFacet` global pointers persist across frames if `r_debugSurface` is enabled, potentially stalling traces when debug visualization is active (they point to the last-hit patch, but remain set even if that patch is no longer traced).
