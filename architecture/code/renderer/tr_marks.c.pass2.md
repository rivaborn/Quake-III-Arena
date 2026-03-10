# code/renderer/tr_marks.c — Enhanced Analysis

## Architectural Role
Implements decal/mark projection as a specialized rendering service integrated into the Renderer DLL but operating off the per-frame critical path. Reuses the BSP tree traversal patterns and surface type dispatch of the main render pipeline (`tr_world.c`, `tr_surface.c`) while operating on impact events from cgame rather than each frame. Results flow back to cgame for visual effect rendering.

## Key Cross-References

### Incoming (who depends on this file)
- **cgame VM** invokes marks projection via `RE_MarkFragments` syscall (exposed through `refexport_t` vtable in renderer public API)
- Result fragments (`pointBuffer`, `markFragment_t[]`) populate cgame's fixed mark pool for decal rendering (`cg_marks.c`)
- No direct runtime dependency from other renderer modules; decals are a separate rendering feature

### Outgoing (what this file depends on)
- **BSP world model:** `tr.world->nodes` (tree root), node/surface linking (built during `RE_LoadWorldMap`)
- **Surface introspection:** examines `msurface_t`, `srfSurfaceFace_t`, `srfGridMesh_t` type discriminators and shader/plane data
- **Deduplication state:** reads/writes `tr.viewCount` (shared with main visibility pass) and `surf->viewCount` (stamps for duplicate tracking)
- **Shared utilities:** `BoxOnPlaneSide` (AABB vs. plane intersection), math ops (`DotProduct`, `CrossProduct`, `VectorNormalize*`), bounds (`ClearBounds`, `AddPointToBounds`)
- **Common layer:** `Com_Memcpy` for bulk point data

## Design Patterns & Rationale

**Sutherland-Hodgman Clipping (1974):** Single-plane polygon clipping (`R_ChopPolyBehindPlane`) is a classical algorithm. Here it clips projected surface triangles against the mark polygon's 6+ bounding planes sequentially. The algorithm naturally handles edge cases (vertices on plane, degenerate output) without special logic.

**Ping-Pong Buffer Alternation:** `clipPoints[2][MAX_VERTS_ON_POLY]` avoids dynamic allocation. The `pingPong` XOR toggle reads from one buffer, writes to the other, and swaps for the next plane. Stack-resident, predictable memory usage—important for a subsystem that may run in interrupt-like conditions (impact handling).

**Deduplication via `tr.viewCount` Stamp:** The main render path increments `tr.viewCount` per frame; here, each mark projection increments it once. The stamp serves dual purposes: (1) prevents surfaces spanning multiple BSP leaves from being clipped multiple times, (2) implicitly decouples mark projection from the main frame's visibility state. However, this means `tr.viewCount` pollution can affect both systems if one stalls.

**Surface-Type Dispatch:** Follows the render backend pattern (examine `*surfaceType_t`) but is selective: only `SF_FACE` (BSP faces via index list) and `SF_GRID` (Bézier patches) are handled. `SF_TRIANGLES` and others are silently ignored—a deliberate design choice, not an oversight. Mirrors cgame's polygon rendering which also doesn't support all types.

**Grid Triangulation at Runtime:** Rather than store precomputed triangle indices, each grid cell is triangulated on-demand. The inline comment explains the rationale: pre-clipping/chopping the grid risked LOD sync problems with the main renderer's runtime LOD application. Current approach is conservative: always use full resolution, accept the per-mark cost.

**Gameplay-Driven Surface Filtering:** Rejection of `SURF_NOIMPACT`, `SURF_NOMARKS`, `CONTENTS_FOG` isn't purely geometric—it's enforced by game rules. Level designers use these flags to protect certain surfaces. The face angle threshold (`DotProduct > -0.5` for SF_FACE, `> -0.1` / `> -0.05` for grid triangles) reflects gameplay intent: don't mark steep or back-facing surfaces where marks would look wrong.

## Data Flow Through This File

```
Entry: cgame impact → RE_MarkFragments(points[], projection, maxPoints, maxFragments)
  ↓
Setup: compute AABB (mark polygon + projection shadow), derive plane equations
  ↓
BSP Traverse: R_BoxSurfaces_r walks tree, collects all surfaces in AABB
  ↓
Per-Surface:
  ├─ Filter by shader flags, face angle, type
  ├─ Clip to projection planes (Sutherland-Hodgman)
  ├─ Store clipped points and fragment metadata
  └─ Repeat for next surface
  ↓
Exit: return fragment count; cgame reads pointBuffer[fragment[i].firstPoint + offset]
```

**Critical State Mutation:**
- `tr.viewCount++` at function start (affects all surface deduplication in the system)
- `surf->viewCount = tr.viewCount` set on every surface touched (marks it as "seen this frame")

## Learning Notes

**Hybrid Classical Algorithm:** Sutherland-Hodgman (1974) in a 2005 engine. No compute shader, no GPU clipping—CPU-side polygon reduction is still practical for marks. Shows that algorithmic fundamentals outlast implementation fads.

**Spatial Index Leverage:** The BSP tree, built once per map load, serves multiple queries: visibility PVS+frustum (`tr_world.c`), dynamic light lists (`tr_light.c`), shadow frustums (`tr_shadows.c`), and here (AABB sweep). A unified spatial structure amortizes construction cost across diverse workloads. Modern engines might use BVH or octree, but BSP was the standard for Q3A-era architecture.

**Off-Path Computation Pattern:** Unlike the render loop (sorted per-frame, frame-to-frame coherence), mark projection is stateless. Results are computed once on impact and immediately consumed. No temporal filtering, no frame-rate dependency. Clean separation of concerns: impact events are asynchronous to rendering.

**Level Designer Intent in Rendering:** Shader flags (`SURF_NOMARKS`) encode design rules in geometry metadata. The renderer doesn't discover these rules; it applies them. Shows Q3A's data-driven architecture where rendering behavior is parameterized by content.

**Grid LOD as Unsolved Design Problem:** The lengthy comment (~30 lines) about grid LOD sync reveals a real tradeoff: chopping the grid to the mark's bounding planes would reduce clipping work, but synchronizing LOD between chopped and original grids is hard. Abandoning that optimization (always triangulate full resolution) suggests pragmatism over perfection.

**Memory Layout Awareness:** Fixed-size buffers (`MAX_VERTS_ON_POLY=64`, surface list `[64]`) are stack-resident and pre-allocated by the caller. No hidden allocations. Predictable memory profile—important for frame-time budgeting.

## Potential Issues

**Silent Surface Truncation:** `R_BoxSurfaces_r` stops at 64 surfaces with no feedback. Large marks over dense geometry lose coverage silently; debug build asserts catch it, but production doesn't. Better: either increase the limit, use dynamic allocation, or return partial results with a "truncated" flag.

**Grid Retriangulation Overhead:** Every mark recomputes grid cell triangles. Large marks over detail geometry (e.g., terrain grids) re-triangulate the same cells multiple times per frame if multiple impacts occur. A precomputed grid triangle index cache could reduce redundant work.

**Threshold Magic Numbers:** Face filtering uses three different angle thresholds (`-0.5`, `-0.1`, `-0.05`) without named constants or comments explaining why. These appear tuned empirically but smell like cargo-cult constants—adjusting one without understanding all three could break expected behavior.

**`MARKER_OFFSET` Dead Code:** The vertex normal-offset path is compiled out (`#define MARKER_OFFSET 0`), yet the code remains. Suggests an incomplete feature (likely added to fix LOD artifacts, then disabled as a workaround). Should either be completed or removed.

**No Frustum Occlusion Cull:** Marks are projected in world space; no check ensures clipped fragments are within the view frustum. For large maps with impacts outside the view, this wastes CPU clipping. cgame's rendering will cull them visually, but the projection cost is already spent. A near-term optimization: pass frustum planes to avoid clipping off-screen geometry.
