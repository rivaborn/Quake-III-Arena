# code/renderer/tr_world.c — Enhanced Analysis

## Architectural Role

This file is the **renderer front-end's BSP world traversal engine**—the linchpin between scene collection and draw surface submission. It bridges two critical subsystem layers: the Renderer (front-end/back-end split) sits above, and qcommon's collision module (CM_ClusterPVS) and math utilities sit below. Every visible world surface in a frame passes through this module exactly once, making it a high-frequency critical path.

## Key Cross-References

### Incoming (who depends on this file)
- **`tr_main.c:RenderScene()`** — calls `R_AddWorldSurfaces()` once per view during front-end scene generation, after `R_MarkMD3Surfaces()` and before `R_AddPolygonSurfaces()`. This is the frame entry point.
- **No other direct callers** — brush model surfaces routed through `R_AddBrushModelSurfaces()` which is called from entity rendering dispatch, also in `tr_main.c`.

### Outgoing (what this file depends on)
- **`tr_main.c`**: `R_CullLocalBox()`, `R_CullPointAndRadius()`, `R_CullLocalPointAndRadius()` (frustum/sphere cull tests), `R_AddDrawSurf()` (submit surfaces to sort list)
- **`tr_light.c`**: `R_DlightBmodel()` (mark dlight bits on inline model)
- **`tr_model.c`**: `R_GetModelByHandle()` (load model from handle)
- **`qcommon/cm_*`**: `CM_ClusterPVS()` (PVS lookup), `R_PointInLeaf()` indirectly uses BSP walk
- **Math/shared**: `DotProduct()`, `BoxOnPlaneSide()`, `ClearBounds()` (both in `code/game/q_math.c`)
- **Platform**: `ri.Error()`, `ri.Printf()` (refimport vtable)

## Design Patterns & Rationale

**Two-Phase Frustum Culling (sphere→AABB)**: `R_CullGrid()` tests a sphere first (cheap), then AABB only if sphere is clipped. This reduces expensive plane tests. Rationale: Bezier patches have well-defined minimum bounding spheres; early rejection saves quad-plane tests.

**Bitmask Dlight Distribution**: Instead of per-surface dlight tests, bits are split at BSP planes (`DotProduct(dl->origin, plane->normal) - plane->dist`) and recursively passed down. This amortizes dlight culling across the BSP tree and avoids redundant per-light-per-surface distance tests. Limited to 32 dlights (one bit per index) — a deliberate constraint for 1999 hardware.

**`viewCount` Frame Marking**: Each surface tracks `surf->viewCount == tr.viewCount`. Since surfaces can span multiple leaves (e.g., large polygons), this guards against duplicate submission. Cheaper than a set lookup; exploits frame-local visibility assumption.

**Tail Recursion → Iteration**: `R_RecursiveWorldNode()` uses a `do/while` loop to convert back-child recursion into iteration (`node = node->children[1]`). Saves stack depth and aligns with stack-constrained 1990s C engines.

**Plane-Based Back-Face Culling**: Uses plane equation with 8-unit epsilon rather than vertex normals. Rationale: BSP planes are precomputed during compilation; epsilon avoids pixel gaps from rounding through BSP→ICD→hardware.

## Data Flow Through This File

**Entry**: `R_AddWorldSurfaces()` receives `tr.viewCluster`, frustum planes, dlight list from `tr.refdef`.

**Phase 1 (Visibility)**: `R_MarkLeaves()` walks PVS from current cluster, stamping `visframe = tr.visCount` on all reachable nodes/leaves.

**Phase 2 (BSP Traversal)**: `R_RecursiveWorldNode()` descends tree:
- Tests node AABB against frustum planes (bitmask tracks which planes still matter)
- At interior nodes: splits dlight bits across plane, recurses both children
- At leaves: expands `visBounds` (used for z-buffer bounds), collects marksurfaces

**Phase 3 (Surface Dispatch)**: `R_AddWorldSurface()` for each marksurface:
- Guards duplication via `viewCount`
- Culls surface (back-face, sphere, AABB, curve toggles)
- Culls dlights per surface (plane distance for faces, AABB for grids)
- Submits to `R_AddDrawSurf()` with shader, fog index, dlight bits

**Exit**: Draw surface list fed to back-end command queue after sorting by shader/fog/entity/dlight.

## Learning Notes

**Idiomatic to the Quake era**: PVS-driven traversal is classic early-2000s engine design. Modern engines prefer occlusion culling (hardware queries or compute), frustum-to-BVH traversal without BSP, or compute-driven cluster culling. Quake's strength was that BSP naturally encodes PVS; the tradeoff is that BSP must be fixed (no dynamic geometry).

**Frame Marking Patterns**: The `viewCount` technique is elegant for single-threaded CPU-driven culling and appears throughout Quake's renderer (`tr_main.c:R_MarkFragments()` uses similar marking). Thread-safe modern engines use generation counters or IDs instead.

**Plane Splits for Lights**: Splitting dlight bits via plane equations (`DotProduct` tests) is computationally cheaper than AABB overlap tests for each light. Reflects 1999 CPU cost models where FPU ops were relatively fast.

**Explicit Surface-Type Dispatch**: Functions like `R_CullSurface()` discriminate on `*surface` (first `int`), requiring manual dispatch to `R_CullGrid()` / `R_CullTriSurf()` / face logic. Modern engines often use vtable pointers or tagged unions; this approach trades indirection for clarity in 1999.

**PVS Assumption**: Entire visibility pipeline assumes world is static. Inline models (brush models) bypass PVS entirely, frustum-culled as opaque boxes. Inferable: entity-bearing models (MD3, MD4) are handled separately in `tr_main.c:R_AddMD3Surfaces()`.

## Potential Issues

1. **32-Dlight Limit**: `dlightBits` is `int` (bitfield). Modern maps may exceed 32 lights. Workaround: `SV_Snapshot.c` clamps `refdef.num_dlights` to 32 before handing to renderer. Not a bug, but a design constraint.

2. **Unimplemented Triangle Culling**: `R_DlightTrisurf()` has marked FIXME; it passes dlights through unconditionally (no AABB test). Inferable: misc_model geometry (triangles) gets full dlight processing in back-end, deferring per-triangle cost to rasterization. OK for typical misc_model sizes (~50 tris), but large models will waste fillrate.

3. **No Dynamic Surface Reordering**: Surfaces are submitted in BSP traversal order, not sorted by material cost. Back-end `R_SortDrawSurfs()` reorders by shader, but spatial clustering is lost. Negligible impact on 1999 GPUs; modern tiling GPUs would benefit from better order.

4. **PVS Carve Errors**: If map BSP is malformed (cluster numbers out of range), `R_ClusterPVS()` returns `novis` (all clusters visible). Silent fallback; errors should be caught at compile time (`q3map`).
