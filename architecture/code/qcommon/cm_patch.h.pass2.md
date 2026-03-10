# code/qcommon/cm_patch.h — Enhanced Analysis

## Architectural Role

This file is part of **qcommon**'s collision detection subsystem, positioned on the critical path for both server-side physics simulation and client-side movement prediction. While **cm_load.c** builds static BSP collision geometry at map load, **cm_patch.h** bridges specialized curved-surface (Bézier patch) collision into the same unified trace/test interface. Every patch surface in a BSP is converted once via `CM_GeneratePatchCollide` during map initialization, then queried per-frame by `CM_Trace` calls originating from the **server frame loop** (authoritative) and **cgame prediction** (speculative client-side), making this data structure foundational to both simulation and client-visible behavior.

## Key Cross-References

### Incoming (who depends on this file)
- **cm_load.c** — parses BSP patch surface records, allocates `patchCollide_t` structures during map load
- **cm_trace.c** / **CM_Trace** — queries `patchCollide_t` arrays during per-frame swept-box and point-content queries
- **server frame loop** (via `SV_GameSystemCalls` / `trap_Trace`) — server-side physics traces against patch collision
- **cgame prediction** (via `trap_Trace` in `cg_predict.c`) — client-side movement prediction traces against patch geometry
- **CM_PositionTestInPatchCollide**, **CM_TraceThroughPatchCollide** (declared in comment) — internal traversal functions that consume `patchCollide_t` directly

### Outgoing (what this file depends on)
- **cm_local.h** — defines `traceWork_t`, used in `CM_TraceThroughPatchCollide` signature
- **q_shared.c** / **q_math.c** — vector utilities (`vec3_t` operations) used during patch point processing
- **cm_polylib.c** / **cm_patch.c** (implementation) — likely use polylib geometry utilities for winding/plane operations during tessellation

## Design Patterns & Rationale

**Two-level hierarchical plane representation:** `patchCollide_t` holds a flat array of `patchPlane_t` records (surface + edge planes) and a parallel `facet_t` array linking facets to plane indices. This avoids deep pointer chasing during collision tests.

**Precomputed `signbits` field:** Each plane stores a bitmask encoding the sign of the plane normal in each axis (`signx + (signy<<1) + (signz<<2)`). This enables O(1) AABB-vs-plane rejection during sweep traces without recomputing plane coefficients—a crucial optimization for 1990s CPU budgets.

**Load-time subdivision strategy:** The `cGrid_t` intermediate structure and `SUBDIVIDE_DISTANCE` constant (16 units) indicate that fine tessellation happens once during map load, trading memory (O(width×height) factets) for per-frame collision speed. This reflects the era's assumption of static geometry and limited VRAM.

**Fixed capacity limits:** `MAX_FACETS 1024` and `MAX_PATCH_PLANES 2048` were likely chosen based on worst-case Q3A test maps and available hunk memory (~8–16 MB total per level). Modern engines use dynamic allocation or streaming.

## Data Flow Through This File

1. **Load phase** (`cm_load.c`): For each patch surface in the BSP, call `CM_GeneratePatchCollide( width, height, points )`.
2. **Tessellation** (inside `cm_patch.c`): Recursively subdivide the control-point grid to `SUBDIVIDE_DISTANCE` tolerance, generating a `cGrid_t` of final vertices.
3. **Plane/facet generation**: Build surface plane equations and edge bevels; compute facet adjacency and inward flags for robust collision response.
4. **Storage** (implicit): Return heap-allocated `patchCollide_t`; pointer stored in BSP surface structure (not visible in this header).
5. **Query phase** (every frame): `CM_TraceThroughPatchCollide` uses plane arrays and facet indices to reject or collide swept boxes against the patch.

## Learning Notes

**Deferred tessellation is the key insight:** Unlike modern real-time ray-marching (GPU-friendly), Q3 pre-tessellates patches into a collision mesh, treating curved surfaces as flat facets. This simplifies the trace algorithm but requires accurate subdivision to avoid missing thin geometry.

**The `borderPlanes` and `borderInward` arrays** (facet_t) encode edge bevels explicitly—a defensive measure to prevent bullets from slipping between facets or getting stuck on seams. Modern collision engines use continuous collision detection (CCD) or analytical swept shapes instead.

**Idiomatic to this era:** Hard-coded plane limits, pre-computed signbits (vs. branch prediction), and load-time amortization—all reflect 1990s CPU constraints and lack of dynamic memory flexibility. Modern engines use BVH trees, implicit surfaces, or GPU acceleration.

**Connection to ECS/scene graph concepts:** This header shows a **component-like pattern**: each patch surface owns a `patchCollide_t` handle, decoupled from rendering or logic. However, it's pre-bound at load, not runtime-swappable like modern ECS.

## Potential Issues

- **No declared ownership model:** `CM_GeneratePatchCollide` returns an opaque pointer; no explicit deallocation signature in the header suggests memory leaks or reliance on `CM_ClearLevelPatches` to bulk-free at map unload.
- **Known geometric edge cases** (per header comment): "Position test of a point against a surface is not well defined, because the surface has no volume." This hints at numerical instability when testing whether a point is inside a patch—relying on conservative mesh structure rather than proper signed-distance semantics.
- **Static-only assumption:** No provisions for animated or deformable patches; incompatible with modern skeletal deformation or real-time CSG.
- **Silent subdivision failure risk:** If a patch's width/height exceeds `MAX_GRID_SIZE (129)`, the header does not warn; `CM_GeneratePatchCollide` likely asserts or silently clips, invisible to callers.
