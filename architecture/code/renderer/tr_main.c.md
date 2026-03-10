# code/renderer/tr_main.c

## File Purpose
This is the main control-flow file for the renderer front end, responsible for per-frame view setup, frustum culling, draw-surface submission and sorting, portal/mirror view recursion, and dispatching entity surfaces to the back-end command queue.

## Core Responsibilities
- Build and set up view-space orientation matrices (`R_RotateForViewer`, `R_RotateForEntity`)
- Compute and set the perspective projection matrix and far-clip distance
- Derive frustum planes for view-space culling
- Perform AABB and sphere frustum culling (`R_CullLocalBox`, `R_CullPointAndRadius`)
- Handle portal and mirror surface detection, orientation computation, and recursive view rendering
- Collect and sort all draw surfaces for a frame (`R_AddDrawSurf`, `R_SortDrawSurfs`)
- Dispatch sorted surfaces to the render back end via `R_AddDrawSurfCmd`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `trGlobals_t` (`tr`) | struct (global instance) | Master renderer state: current entity, view parms, refdef, models, shaders, images |
| `viewParms_t` | struct | Per-view state: orientation, frustum, projection matrix, viewport, zFar, portal flags |
| `orientationr_t` | struct | Local-to-world transform: origin, axis[3], viewOrigin, modelMatrix[16] |
| `drawSurf_t` | struct | Packed sort key + surface pointer submitted to the back end |
| `orientation_t` | struct | Lightweight origin + axis used for portal/mirror math |
| `surfaceType_t` | enum/typedef | Tag at the head of every surface structure, drives dispatch table |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tr` | `trGlobals_t` | global | Central renderer globals; read/written throughout the front end |
| `ri` | `refimport_t` | global | Engine import table (Printf, Error, CM callbacks, etc.) |
| `entitySurface` | `surfaceType_t` | global | Sentinel surface value used for procedurally generated entity surfaces |
| `s_flipMatrix` | `float[16]` | static | Constant rotation from Quake (X-forward) to OpenGL (-Z-forward) coordinate space |

## Key Functions / Methods

### R_CullLocalBox
- **Signature:** `int R_CullLocalBox(vec3_t bounds[2])`
- **Purpose:** AABB frustum cull test in world space; transforms 8 box corners and tests against 4 frustum planes.
- **Inputs:** `bounds[2]` — local-space min/max corners.
- **Outputs/Return:** `CULL_IN`, `CULL_CLIP`, or `CULL_OUT`.
- **Side effects:** Reads `tr.or`, `tr.viewParms.frustum`, `r_nocull`.
- **Calls:** `VectorCopy`, `VectorMA`, `DotProduct`.
- **Notes:** Short-circuits to `CULL_CLIP` when `r_nocull` is set.

### R_CullPointAndRadius
- **Signature:** `int R_CullPointAndRadius(vec3_t pt, float radius)`
- **Purpose:** Sphere frustum cull test against the 4 view planes.
- **Inputs:** World-space point, bounding sphere radius.
- **Outputs/Return:** `CULL_IN`, `CULL_CLIP`, or `CULL_OUT`.
- **Side effects:** Reads `tr.viewParms.frustum`, `r_nocull`.
- **Calls:** `DotProduct`.

### R_RotateForEntity
- **Signature:** `void R_RotateForEntity(const trRefEntity_t *ent, const viewParms_t *viewParms, orientationr_t *or)`
- **Purpose:** Builds the local-to-clip model matrix for an entity and computes `viewOrigin` in model space (used for fog, specular, env-mapping).
- **Inputs:** Entity, current view parms.
- **Outputs/Return:** Fills `*or` in-place (no GL calls).
- **Side effects:** None beyond writing `*or`.
- **Calls:** `myGlMultMatrix`, `VectorSubtract`, `VectorLength`, `DotProduct`.
- **Notes:** Non-`RT_MODEL` entities fall back to world orientation; handles non-normalized axes via `axisLength` scale compensation.

### R_RotateForViewer
- **Signature:** `void R_RotateForViewer(void)`
- **Purpose:** Constructs the camera view matrix from `tr.viewParms.or`, applies the Quake→OpenGL flip, and stores the result as `tr.or.modelMatrix` / `tr.viewParms.world`.
- **Inputs:** Reads `tr.viewParms`.
- **Outputs/Return:** Writes `tr.or`, `tr.viewParms.world`.
- **Side effects:** Mutates `tr.or` (global front-end state).
- **Calls:** `myGlMultMatrix`.

### R_SetupProjection
- **Signature:** `void R_SetupProjection(void)`
- **Purpose:** Computes the OpenGL perspective matrix from `fov_x/fov_y` and near/far clip values, storing it in `tr.viewParms.projectionMatrix`.
- **Side effects:** Calls `SetFarClip()` (updates `tr.viewParms.zFar`); writes `tr.viewParms.projectionMatrix`.
- **Calls:** `SetFarClip`, `tan`, `sqrt` (inside `SetFarClip`).

### R_SetupFrustum
- **Signature:** `void R_SetupFrustum(void)`
- **Purpose:** Derives 4 frustum planes (left, right, bottom, top) from the view axis and half-angle trig.
- **Side effects:** Writes `tr.viewParms.frustum[0..3]`, calls `SetPlaneSignbits`.
- **Calls:** `sin`, `cos`, `VectorScale`, `VectorMA`, `DotProduct`, `SetPlaneSignbits`.

### R_GetPortalOrientations
- **Signature:** `qboolean R_GetPortalOrientations(drawSurf_t*, int entityNum, orientation_t *surface, orientation_t *camera, vec3_t pvsOrigin, qboolean *mirror)`
- **Purpose:** Locates the portal entity matching a portal surface and computes both surface and camera orientations, including optional animated rotation.
- **Outputs/Return:** `qtrue` if a valid portal entity was found; fills `surface`, `camera`, `pvsOrigin`, `*mirror`.
- **Side effects:** Mutates `tr.currentEntityNum`, `tr.currentEntity`, `tr.or` temporarily.
- **Calls:** `R_PlaneForSurface`, `R_RotateForEntity`, `R_LocalNormalToWorld`, `PerpendicularVector`, `CrossProduct`, `RotatePointAroundVector`.

### R_MirrorViewBySurface
- **Signature:** `qboolean R_MirrorViewBySurface(drawSurf_t *drawSurf, int entityNum)`
- **Purpose:** Entry point for recursive mirror/portal rendering; validates visibility, builds new `viewParms_t`, and calls `R_RenderView` recursively.
- **Outputs/Return:** `qtrue` if a recursive view was rendered.
- **Side effects:** Calls `R_RenderView` (recursive); temporarily modifies `tr.viewParms`.
- **Calls:** `SurfIsOffscreen`, `R_GetPortalOrientations`, `R_MirrorPoint`, `R_MirrorVector`, `R_RenderView`.
- **Notes:** Guards against infinite recursion via `tr.viewParms.isPortal`; also gated by `r_noportals` and `r_fastsky`.

### R_SortDrawSurfs
- **Signature:** `void R_SortDrawSurfs(drawSurf_t *drawSurfs, int numDrawSurfs)`
- **Purpose:** Sorts the draw-surface list by packed sort key, scans for portal surfaces that must be rendered first, then enqueues to back end.
- **Side effects:** Calls `R_MirrorViewBySurface` for portal shaders; calls `R_AddDrawSurfCmd`.
- **Calls:** `qsortFast`, `R_DecomposeSort`, `R_MirrorViewBySurface`, `R_AddDrawSurfCmd`.

### R_AddEntitySurfaces
- **Signature:** `void R_AddEntitySurfaces(void)`
- **Purpose:** Iterates all entities in the refdef, culls first-person entities in portal views, and dispatches to type-specific surface adders (`R_AddMD3Surfaces`, `R_AddAnimSurfaces`, `R_AddBrushModelSurfaces`).
- **Side effects:** Writes `tr.currentEntityNum`, `tr.shiftedEntityNum`, `tr.currentEntity`, `tr.currentModel`, `tr.or`.
- **Calls:** `R_RotateForEntity`, `R_GetModelByHandle`, `R_GetShaderByHandle`, `R_SpriteFogNum`, `R_AddDrawSurf`, `R_AddMD3Surfaces`, `R_AddAnimSurfaces`, `R_AddBrushModelSurfaces`.

### R_RenderView
- **Signature:** `void R_RenderView(viewParms_t *parms)`
- **Purpose:** Top-level per-view render driver; increments counters, sets up matrices, generates and sorts draw surfaces, and fires debug visualization.
- **Side effects:** Increments `tr.viewCount`; mutates `tr.viewParms`, `tr.or`; enqueues draw commands.
- **Calls:** `R_RotateForViewer`, `R_SetupFrustum`, `R_GenerateDrawSurfs`, `R_SortDrawSurfs`, `R_DebugGraphics`.

## Control Flow Notes
`R_RenderView` is the per-view entry point, called by `RE_RenderScene` for the primary view and recursively by `R_MirrorViewBySurface` for portals (guarded by `isPortal` flag). Within a view: `R_RotateForViewer` → `R_SetupFrustum` → `R_GenerateDrawSurfs` (world + polygons + projection + entities) → `R_SortDrawSurfs` → `R_AddDrawSurfCmd` (hand-off to back end). Portal surfaces trigger a sub-call to `R_RenderView` before the main surface list is submitted.

## External Dependencies
- **`tr_local.h`** — all renderer types, cvar externs, and subsystem prototypes
- **`q_shared.h` / `qcommon.h`** — math primitives (`VectorMA`, `DotProduct`, `PlaneFromPoints`, `PerpendicularVector`, `CrossProduct`, `RotatePointAroundVector`, `SetPlaneSignbits`)
- **`RB_BeginSurface`, `rb_surfaceTable`** — back-end surface tessellation (defined in `tr_backend.c` / `tr_surface.c`)
- **`R_AddWorldSurfaces`, `R_AddPolygonSurfaces`** — world and polygon surface adders (defined in `tr_world.c` / `tr_scene.c`)
- **`R_AddMD3Surfaces`, `R_AddAnimSurfaces`, `R_AddBrushModelSurfaces`** — model-type-specific surface adders (defined elsewhere in renderer)
- **`R_AddDrawSurfCmd`** — enqueues the sorted surface list to the back-end command buffer (defined in `tr_cmds.c`)
- **`R_SyncRenderThread`** — SMP render-thread synchronization (defined in `tr_init.c` or platform layer)
- **`ri.CM_DrawDebugSurface`** — collision-map debug callback (defined in collision module)
- **`tess`** (`shaderCommands_t`) — global tessellator state (defined in `tr_shade.c`)
