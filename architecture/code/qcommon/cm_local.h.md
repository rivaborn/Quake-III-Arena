# code/qcommon/cm_local.h

## File Purpose
Private header for the collision map (CM) subsystem, defining all internal data structures, the global clip map state, and declaring the internal API shared across `cm_load.c`, `cm_test.c`, `cm_trace.c`, and `cm_patch.c`. It is never included by code outside the `qcommon/cm_*` family.

## Core Responsibilities
- Define the in-memory BSP collision tree types (`cNode_t`, `cLeaf_t`, `cmodel_t`, `cbrush_t`, `cPatch_t`, `cArea_t`)
- Define the monolithic `clipMap_t` structure that holds the entire loaded collision world
- Declare the global `cm` instance and all CM-related debug/trace counters
- Define the per-trace working state (`traceWork_t`, `sphere_t`) used during box/capsule sweeps
- Declare the leaf-enumeration utility type (`leafList_t`) and its callbacks
- Declare internal cross-file functions for box queries, leaf enumeration, and patch collision

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cNode_t` | struct | BSP tree internal node: plane pointer + two children indices |
| `cLeaf_t` | struct | BSP leaf: cluster/area ID, references into leaf-brush and leaf-surface index arrays |
| `cmodel_t` | struct | Inline submodel: AABB + embedded `cLeaf_t` (submodels bypass the main tree) |
| `cbrushside_t` | struct | One side of a brush: plane, surface flags, shader index |
| `cbrush_t` | struct | Convex brush: shader, contents mask, AABB, sides array, check-count stamp |
| `cPatch_t` | struct | Curved surface patch proxy: check-count stamp, flags, pointer to `patchCollide_s` |
| `cArea_t` | struct | Flood-fill area: flood number and validity stamp for area-portal queries |
| `clipMap_t` | struct | Top-level collision world: owns all planes, nodes, leafs, brushes, patches, visibility, entity string, and area portals |
| `sphere_t` | struct | Oriented capsule parameters (radius, half-height, offset) for capsule traces |
| `traceWork_t` | struct | Per-trace scratch: start/end, swept box corners (8 offsets), extents, enclosing bounds, model origin, contents, result `trace_t`, and optional `sphere_t` |
| `leafList_t` | struct | Accumulator for `CM_BoxLeafnums_r`: bounded list with overflow flag and a pluggable `storeLeafs` callback |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cm` | `clipMap_t` | global (extern) | The single loaded collision world |
| `c_pointcontents` | `int` | global (extern) | Debug counter: point-in-contents tests |
| `c_traces` | `int` | global (extern) | Debug counter: total traces |
| `c_brush_traces` | `int` | global (extern) | Debug counter: brush-specific traces |
| `c_patch_traces` | `int` | global (extern) | Debug counter: patch-specific traces |
| `cm_noAreas` | `cvar_t *` | global (extern) | CVar: disables area-portal flood-fill |
| `cm_noCurves` | `cvar_t *` | global (extern) | CVar: disables curved-surface collision |
| `cm_playerCurveClip` | `cvar_t *` | global (extern) | CVar: toggles player-vs-curve clipping |

## Key Functions / Methods

### CM_BoxBrushes
- **Signature:** `int CM_BoxBrushes( const vec3_t mins, const vec3_t maxs, cbrush_t **list, int listsize )`
- **Purpose:** Returns all brushes whose AABBs overlap the given box.
- **Inputs:** World-space AABB `mins`/`maxs`; output array `list` of capacity `listsize`.
- **Outputs/Return:** Count of overlapping brushes; fills `list`.
- **Side effects:** None inferable beyond reading `cm`.
- **Calls:** Defined in `cm_test.c`; internally calls `CM_BoxLeafnums_r`.
- **Notes:** Used for quick broad-phase brush queries (e.g., bot code).

### CM_StoreLeafs / CM_StoreBrushes
- **Signature:** `void CM_StoreLeafs( leafList_t *ll, int nodenum )` / `void CM_StoreBrushes( leafList_t *ll, int nodenum )`
- **Purpose:** Pluggable `storeLeafs` callbacks for `leafList_t`; accumulate leaf indices or brush pointers during BSP traversal.
- **Inputs:** `ll` — accumulator; `nodenum` — current BSP node/leaf.
- **Outputs/Return:** Void; writes into `ll->list`.
- **Side effects:** Sets `ll->overflowed` if capacity exceeded.

### CM_BoxLeafnums_r
- **Signature:** `void CM_BoxLeafnums_r( leafList_t *ll, int nodenum )`
- **Purpose:** Recursive BSP descent collecting all leafs that overlap `ll->bounds`.
- **Inputs:** `ll` with bounds and callback set; starting `nodenum`.
- **Side effects:** Calls `ll->storeLeafs` for each intersecting leaf.

### CM_ClipHandleToModel
- **Signature:** `cmodel_t *CM_ClipHandleToModel( clipHandle_t handle )`
- **Purpose:** Maps a public `clipHandle_t` (index) to the internal `cmodel_t` pointer; handles special handles `BOX_MODEL_HANDLE` (255) and `CAPSULE_MODEL_HANDLE` (254).
- **Outputs/Return:** Pointer into `cm.cmodels[]` or a special synthetic model.

### CM_GeneratePatchCollide
- **Signature:** `struct patchCollide_s *CM_GeneratePatchCollide( int width, int height, vec3_t *points )`
- **Purpose:** Builds a `patchCollide_s` collision structure from a grid of control points (Bezier patch).
- **Inputs:** Grid dimensions and point array.
- **Outputs/Return:** Heap-allocated `patchCollide_s`; caller owns it via `cm`.
- **Side effects:** Allocates memory.

### CM_TraceThroughPatchCollide
- **Signature:** `void CM_TraceThroughPatchCollide( traceWork_t *tw, const struct patchCollide_s *pc )`
- **Purpose:** Clips a swept box/capsule trace against a patch collision structure; updates `tw->trace`.
- **Side effects:** Modifies `tw->trace.fraction`, `plane`, `surfaceFlags`, `contents`.

### CM_PositionTestInPatchCollide
- **Signature:** `qboolean CM_PositionTestInPatchCollide( traceWork_t *tw, const struct patchCollide_s *pc )`
- **Purpose:** Tests whether a stationary box overlaps a patch (used for `startsolid` detection).
- **Outputs/Return:** `qtrue` if intersecting.

### CM_ClearLevelPatches
- **Signature:** `void CM_ClearLevelPatches( void )`
- **Purpose:** Frees all `patchCollide_s` structures; called on map unload.
- **Side effects:** Frees heap memory; zeroes `cPatch_t` pointers in `cm.surfaces`.

## Control Flow Notes
This header is included at map load time (`cm_load.c` populates `cm`) and remains valid for the duration of a map session. During each frame, `cm_trace.c` fills a stack-allocated `traceWork_t`, traverses `cm`'s BSP tree, and tests against `cbrush_t`/`cPatch_t` entries. On map change, `CM_ClearLevelPatches` and the load path reinitialize `cm`. There is no per-frame init/shutdown; the system is passive (queried by server, prediction, and physics code).

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `cplane_t`, `trace_t`, `clipHandle_t`, `qboolean`, `dshader_t`
- `qcommon.h` → `cm_public.h` — public CM API types
- `cm_polylib.h` — `winding_t` (used only by debug visualization in `cm_debug`)
- `patchCollide_s` — defined in `cm_patch.c` (forward-declared here as an incomplete struct)
- `dshader_t` — defined in `qfiles.h` (BSP on-disk shader lump entry)
