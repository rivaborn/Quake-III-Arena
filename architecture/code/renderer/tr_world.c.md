# code/renderer/tr_world.c

## File Purpose
Implements the renderer front-end world traversal for Quake III Arena. It walks the BSP tree to determine which world surfaces are potentially visible this frame, culls them against the view frustum and PVS, and submits them to the draw surface list. It also handles brush model surface submission and dynamic light (dlight) intersection testing.

## Core Responsibilities
- Mark visible BSP leaves via PVS/areamask (`R_MarkLeaves`)
- Recursively traverse the BSP tree with frustum culling (`R_RecursiveWorldNode`)
- Cull individual surfaces (face, grid, triangle) before submission
- Distribute dlight bits down the BSP tree and per-surface
- Submit visible surfaces to the renderer sort list via `R_AddDrawSurf`
- Handle brush model (inline model) surface submission separately
- Provide `R_inPVS` utility for visibility queries between two points

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `mnode_t` | struct | BSP tree node/leaf; holds frustum bounds, PVS visframe, plane, children, and marksurface lists |
| `msurface_t` | struct | A single world surface with shader, fog index, and typed surface data pointer |
| `srfSurfaceFace_t` | struct | Planar polygon surface; stores plane, dlight bits, vertices/indices |
| `srfGridMesh_t` | struct | Bezier patch grid surface; stores bounds, sphere, dlight bits |
| `srfTriangles_t` | struct | Triangle soup surface (misc_model geometry); bounds and dlight bits |
| `world_t` | struct | Loaded BSP world; nodes, surfaces, PVS data, clusters, fogs |
| `bmodel_t` | struct | Inline brush model; bounds and surface range |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tr` | `trGlobals_t` | global (extern) | Central renderer state: world pointer, view counts, dlight list, perf counters |
| `backEnd` | `backEndState_t` | global (extern) | Back-end state; `smpFrame` used for double-buffered dlight bits |

## Key Functions / Methods

### R_CullTriSurf
- **Signature:** `static qboolean R_CullTriSurf( srfTriangles_t *cv )`
- **Purpose:** Cull a triangle-soup surface using its AABB.
- **Inputs:** Pointer to `srfTriangles_t` with `bounds`.
- **Outputs/Return:** `qtrue` if fully outside frustum.
- **Side effects:** None.
- **Calls:** `R_CullLocalBox`

### R_CullGrid
- **Signature:** `static qboolean R_CullGrid( srfGridMesh_t *cv )`
- **Purpose:** Two-phase cull (sphere then AABB) for Bezier patch grids; respects `r_nocurves`.
- **Inputs:** Pointer to `srfGridMesh_t`.
- **Outputs/Return:** `qtrue` if culled.
- **Side effects:** Increments `tr.pc` sphere/box cull counters.
- **Calls:** `R_CullLocalPointAndRadius`, `R_CullPointAndRadius`, `R_CullLocalBox`
- **Notes:** Uses `tr.currentEntityNum` to choose local vs. world-space sphere test.

### R_CullSurface
- **Signature:** `static qboolean R_CullSurface( surfaceType_t *surface, shader_t *shader )`
- **Purpose:** Dispatch culling by surface type; additionally does back-face culling on `SF_FACE` surfaces using view-plane dot product with an 8-unit epsilon.
- **Inputs:** Generic surface data pointer (discriminated by first `int`), associated shader.
- **Outputs/Return:** `qtrue` if surface should be skipped.
- **Calls:** `R_CullGrid`, `R_CullTriSurf`, `DotProduct`
- **Notes:** Respects `CT_TWO_SIDED`, `CT_FRONT_SIDED`, `CT_BACK_SIDED`; `r_nocull` bypasses all culling.

### R_DlightFace / R_DlightGrid / R_DlightTrisurf
- **Purpose:** Per-surface dlight culling. Removes dlight bits where the light cannot reach the surface plane or AABB. Stores surviving bits into `dlightBits[tr.smpFrame]`.
- **Notes:** `R_DlightTrisurf` has no AABB test (marked FIXME); it passes bits through unconditionally.

### R_DlightSurface
- **Signature:** `static int R_DlightSurface( msurface_t *surf, int dlightBits )`
- **Purpose:** Dispatch dlight culling to the correct per-type helper; updates `tr.pc.c_dlightSurfaces`.
- **Calls:** `R_DlightFace`, `R_DlightGrid`, `R_DlightTrisurf`

### R_AddWorldSurface
- **Signature:** `static void R_AddWorldSurface( msurface_t *surf, int dlightBits )`
- **Purpose:** Guards against duplicate submission (via `viewCount`), culls, resolves dlights, and enqueues the surface.
- **Side effects:** Sets `surf->viewCount`; calls `R_AddDrawSurf` (modifies draw surface list).
- **Calls:** `R_CullSurface`, `R_DlightSurface`, `R_AddDrawSurf`

### R_AddBrushModelSurfaces
- **Signature:** `void R_AddBrushModelSurfaces( trRefEntity_t *ent )`
- **Purpose:** Culls an inline brush model as a whole box, then submits each of its surfaces.
- **Side effects:** Calls `R_DlightBmodel` to set dlight flags on the entity.
- **Calls:** `R_GetModelByHandle`, `R_CullLocalBox`, `R_DlightBmodel`, `R_AddWorldSurface`

### R_RecursiveWorldNode
- **Signature:** `static void R_RecursiveWorldNode( mnode_t *node, int planeBits, int dlightBits )`
- **Purpose:** Core BSP traversal. At each node: checks `visframe`, tests node AABB against active frustum planes (bitmask optimization to skip already-confirmed planes), splits dlight bits across the splitting plane, recurses front child, then tail-recurses into back child. At leaves: expands `visBounds` and submits all mark surfaces.
- **Side effects:** Updates `tr.viewParms.visBounds`, `tr.pc.c_leafs`, and indirectly the draw surface list.
- **Calls:** `BoxOnPlaneSide`, `DotProduct`, `R_RecursiveWorldNode` (recursive), `R_AddWorldSurface`
- **Notes:** Uses a `do/while` loop to convert tail recursion into iteration for the back child. `planeBits` tracks which of the 4 frustum planes still need testing.

### R_PointInLeaf
- **Signature:** `static mnode_t *R_PointInLeaf( const vec3_t p )`
- **Purpose:** Walk BSP tree to find the leaf containing point `p`.
- **Calls:** `DotProduct`; errors via `ri.Error` if world is null.

### R_ClusterPVS
- **Signature:** `static const byte *R_ClusterPVS( int cluster )`
- **Purpose:** Return pointer to the raw PVS bitset for a given cluster; returns `novis` on invalid input.

### R_inPVS
- **Signature:** `qboolean R_inPVS( const vec3_t p1, const vec3_t p2 )`
- **Purpose:** Public API — returns whether `p2` is in the PVS of the leaf containing `p1`.
- **Calls:** `R_PointInLeaf`, `CM_ClusterPVS`

### R_MarkLeaves
- **Signature:** `static void R_MarkLeaves( void )`
- **Purpose:** Determine the current view cluster, then mark all BSP nodes/leaves reachable via PVS and areamask by stamping `visframe = tr.visCount`. Early-outs if cluster and areamask are unchanged.
- **Side effects:** Increments `tr.visCount`, sets `tr.viewCluster`, writes `visframe` on nodes; `r_novis` marks all non-solid nodes visible.
- **Calls:** `R_PointInLeaf`, `R_ClusterPVS`, `ri.Printf`

### R_AddWorldSurfaces
- **Signature:** `void R_AddWorldSurfaces( void )`
- **Purpose:** Frame entry point for world rendering. Guards `r_drawworld` / `RDF_NOWORLDMODEL`, sets entity context, calls `R_MarkLeaves`, clears `visBounds`, clamps dlight count to 32, then kicks off `R_RecursiveWorldNode` with all 4 frustum planes active and all dlight bits set.
- **Calls:** `R_MarkLeaves`, `ClearBounds`, `R_RecursiveWorldNode`

## Control Flow Notes
`R_AddWorldSurfaces` is called once per view from the renderer front-end (scene generation phase, before back-end execution). It is the world equivalent of `R_AddMD3Surfaces`. The function feeds `R_AddDrawSurf` which populates the `drawSurfs` list sorted later for back-end rendering. `R_AddBrushModelSurfaces` is called per-entity for inline models and follows the same path through `R_AddWorldSurface`.

## External Dependencies
- **`tr_local.h`** — all renderer types, globals (`tr`, `backEnd`), cvars, and function prototypes
- **Defined elsewhere:**
  - `R_CullLocalBox`, `R_CullPointAndRadius`, `R_CullLocalPointAndRadius` — `tr_main.c`
  - `R_AddDrawSurf` — `tr_main.c`
  - `R_DlightBmodel` — `tr_light.c`
  - `R_GetModelByHandle` — `tr_model.c`
  - `BoxOnPlaneSide`, `DotProduct`, `ClearBounds` — math/shared utilities
  - `CM_ClusterPVS` — collision map module (`qcommon`)
  - `ri.Error`, `ri.Printf` — platform import table
