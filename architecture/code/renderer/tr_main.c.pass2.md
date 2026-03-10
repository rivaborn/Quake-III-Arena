# code/renderer/tr_main.c — Enhanced Analysis

## Architectural Role

`tr_main.c` is the **orchestration spine of the renderer front end**: it owns per-view setup (matrices, frustum, far clip), all coordinate-space transform utilities consumed engine-wide, the draw-surface sort/dispatch pipeline, and recursive portal/mirror view rendering. It sits directly between `tr_scene.c` (which receives `RE_RenderScene` calls from the client layer) and `tr_backend.c`/`tr_cmds.c` (which execute OpenGL commands). Every other renderer file that culls, transforms, or submits geometry ultimately flows through data structures set up here. The global `trGlobals_t tr` and `viewParms_t tr.viewParms` defined/mutated here are the shared scratchpad that all renderer subsystems read each frame.

## Key Cross-References

### Incoming (who depends on this file)

- **`tr_scene.c` (`RE_RenderScene`)** — sole external caller of `R_RenderView`; also calls `R_AddDrawSurf` indirectly through polygon adders
- **`tr_world.c`** — calls `R_CullLocalBox`, `R_CullPointAndRadius` for BSP leaf/surface culling; reads `tr.viewParms.frustum` set here
- **`tr_model.c`, `tr_mesh.c`, `tr_animation.c`** — call `R_RotateForEntity` and the local↔world transform utilities (`R_LocalPointToWorld`, `R_LocalNormalToWorld`, `R_WorldToLocal`) for MD3/MD4 and brush model transforms
- **`tr_flares.c`, `tr_marks.c`, `tr_light.c`** — call `R_CullPointAndRadius`/`R_CullLocalBox` and `R_TransformModelToClip`/`R_TransformClipToWindow` for flare occlusion and mark placement
- **`tr_shade.c`, `tr_shade_calc.c`** — read `tr.or.viewOrigin` (set by `R_RotateForEntity`) for specular, environment mapping, and fog distance
- **`tr_backend.c`** — consumes draw-command buffers populated by `R_AddDrawSurfCmd` (called from `R_SortDrawSurfs` here); reads `tr.viewParms.projectionMatrix`
- **`entitySurface` global** — referenced by entity surface submission throughout `R_AddEntitySurfaces` and used as a dummy surface pointer across model-adder files

### Outgoing (what this file depends on)

- **`tr_cmds.c`** (`R_AddDrawSurfCmd`, `R_SyncRenderThread`) — hand-off point to the back-end command queue; `R_SyncRenderThread` is critical for SMP correctness before modifying shared command buffers
- **`tr_world.c`** (`R_AddWorldSurfaces`) and **`tr_scene.c`** (`R_AddPolygonSurfaces`) — called from `R_GenerateDrawSurfs`
- **`tr_model.c`** (`R_AddMD3Surfaces`, `R_AddBrushModelSurfaces`) and **`tr_animation.c`** (`R_AddAnimSurfaces`) — called from `R_AddEntitySurfaces`
- **`tr_surface.c`** (`rb_surfaceTable`) — portal plane extraction via `SurfIsOffscreen` and `R_PlaneForSurface`
- **`qcommon` collision module** (`ri.CM_DrawDebugSurface`) — debug overlay injected via `refimport_t ri`; represents the only collision→renderer callback
- **`q_shared.h` / `q_math.c`** — `VectorMA`, `DotProduct`, `CrossProduct`, `PerpendicularVector`, `RotatePointAroundVector`, `SetPlaneSignbits` used pervasively

## Design Patterns & Rationale

- **Immediate-mode sort-and-submit**: draw surfaces are accumulated into `tr.refdef.drawSurfs[]` as packed 64-bit sort keys during scene traversal, then `qsortFast`'d and submitted as a single batch. This avoids per-surface GL state changes and enables shader-ordered rendering — the dominant visual quality technique of 1999-era multi-pass shaders.
- **Dual-coordinate system with explicit flip matrix**: Quake's world space uses X-forward/Y-left/Z-up while OpenGL expects -Z-forward. Rather than adjusting all geometry, a single `s_flipMatrix` bakes the conversion into the view matrix once. This is the cleanest possible fix for a coordinate mismatch that would otherwise infect every transform.
- **`orientationr_t` as per-entity transform scratchpad**: rather than pushing/popping OpenGL matrix state, the front end maintains its own `tr.or` struct that is rebuilt per-entity, decoupling CPU-side transform math from GPU-side state and enabling the SMP split.
- **Single-depth portal recursion**: `isPortal` flag in `viewParms_t` prevents portals-within-portals. This is a deliberate performance/complexity tradeoff — infinite mirror regress is impossible but nested portal rooms are not supported.
- **Procedural surface sentinel** (`entitySurface = SF_ENTITY`): entities that have no real BSP surface (sprites, beams) are given a pointer to this global as their sort key's surface, enabling them to pass through the unified sort pipeline without special-casing the submission path.

## Data Flow Through This File

```
RE_RenderScene (tr_scene.c)
  └─► R_RenderView(viewParms_t*)
        ├─ R_RotateForViewer()         [viewParms.or → tr.or.modelMatrix + s_flipMatrix]
        ├─ R_SetupProjection()
        │    └─ SetFarClip()           [visBounds corners → tr.viewParms.zFar]
        ├─ R_SetupFrustum()            [axis + FOV trig → tr.viewParms.frustum[4]]
        ├─ R_GenerateDrawSurfs()
        │    ├─ R_AddWorldSurfaces()   [BSP PVS traversal → R_AddDrawSurf calls]
        │    ├─ R_AddPolygonSurfaces()
        │    └─ R_AddEntitySurfaces()
        │         └─ (per entity) R_RotateForEntity() → model-specific adder
        └─ R_SortDrawSurfs(drawSurfs, n)
              ├─ qsortFast()
              ├─ (portal shaders) R_MirrorViewBySurface() → R_RenderView() [recursive]
              └─ R_AddDrawSurfCmd()    [→ tr_cmds.c back-end queue]
```

Key state transitions: `tr.or` is clobbered and restored for each entity during `R_AddEntitySurfaces` — callers downstream (shade/fog) must only read `tr.or` while the correct entity is active, which is enforced by the sequential entity loop. `tr.viewParms` is snapshotted into local variables before portal recursion so the parent view is correctly restored.

## Learning Notes

- **Why hand-rolled matrix multiply (`myGlMultMatrix`)?** In 1999, `glGetFloatv(GL_MODELVIEW_MATRIX)` round-trips through the driver; keeping the CPU-side copy avoids that cost and enables SMP (back end never touches CPU-side matrices).
- **Dynamic far clip via `visBounds`**: Rather than a fixed far plane (which wastes depth buffer precision on small maps or wastes nothing on huge ones), Q3 computes the bounding box of all visible BSP leaves each frame and sets `zFar` to the farthest corner distance. This was novel at the time and improves depth precision significantly.
- **No scene graph, no ECS**: entities are a flat `refEntity_t` array submitted by cgame each frame. The renderer owns no entity lifetime; it is purely a one-frame-at-a-time consumer. Modern engines (Unity, Unreal) maintain persistent scene nodes — Q3's approach trades scene management complexity for simplicity and cache efficiency on small entity counts.
- **`r_nocull` cvar short-circuits all culling to `CULL_CLIP`**: useful for debugging but note it returns `CULL_CLIP` not `CULL_IN`, meaning intersection code paths always run — a deliberate choice to stress the back end.
- **Coordinate system archaeology**: the comment "looking down X" vs "looking down -Z" in `s_flipMatrix` reflects Quake's Doom heritage (maps authored with X as forward). Modern engines uniformly adopt OpenGL/Vulkan conventions.

## Potential Issues

- **Portal recursion is depth-1 only**: the `isPortal` flag blocks any second-level portal within a portal view. Attempting stacked mirrors (e.g., mirror facing a portal) silently produces the wrong result (falls through to `r_noportals` path) rather than erroring.
- **`tr.or` is a mutable global mutated mid-traversal**: `R_AddEntitySurfaces` loops through all entities and calls `R_RotateForEntity` into `tr.or` before each adder. Any function called from the adder that reads `tr.or` must be called within that loop iteration — there is no protection against accidental use of a stale `tr.or` outside the loop.
- **`myGlMultMatrix` writes to `out` which may alias `a` or `b`**: no aliasing check or temporary buffer is used. Callers in `R_RotateForEntity` pass a local `glMatrix` for `a` and `or->modelMatrix` for `out` (disjoint), but a future caller could silently corrupt results.
