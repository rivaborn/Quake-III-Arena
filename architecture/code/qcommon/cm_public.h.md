# code/qcommon/cm_public.h

## File Purpose
Public API header for Quake III Arena's collision map (CM) subsystem. It declares all externally-visible functions that other engine modules (server, client, game) use to query the BSP collision world: map loading, spatial queries, trace/sweep tests, PVS lookups, and area portal management.

## Core Responsibilities
- Declare the map load/unload lifecycle interface (`CM_LoadMap`, `CM_ClearMap`)
- Expose clip handle acquisition for inline BSP models and temporary box models
- Provide point and box content queries (brush contents flags)
- Declare box/capsule trace (sweep) functions against the collision world
- Export PVS (Potential Visibility Set) cluster queries
- Expose leaf/area/portal connectivity queries
- Declare tag interpolation, mark fragment, and debug surface utilities

## Key Types / Data Structures
None defined in this file. All types are defined elsewhere and used by reference.

| Name | Kind | Purpose |
|---|---|---|
| `clipHandle_t` | typedef (defined elsewhere) | Opaque handle referencing a collision model or inline BSP submodel |
| `trace_t` | struct (defined elsewhere) | Output record for a box/capsule trace: fraction, endpos, plane, contents, etc. |
| `orientation_t` | struct (defined elsewhere) | Tag orientation result: origin + axes |
| `markFragment_t` | struct (defined elsewhere) | Output record for a decal/mark fragment clipped to world geometry |
| `vec3_t` | typedef (defined elsewhere) | 3-component float vector |

## Global / File-Static State
None. This is a pure declaration header.

## Key Functions / Methods

### CM_LoadMap
- Signature: `void CM_LoadMap(const char *name, qboolean clientload, int *checksum)`
- Purpose: Loads a BSP map file into the collision system, building internal structures.
- Inputs: `name` — map path; `clientload` — whether this is a client-side load (may skip some data); `checksum` — out param for map integrity check.
- Outputs/Return: void; writes checksum via pointer.
- Side effects: Allocates internal CM world state; replaces any previously loaded map.
- Calls: Not inferable from this file.
- Notes: Must be called before any other CM query.

### CM_InlineModel
- Signature: `clipHandle_t CM_InlineModel(int index)`
- Purpose: Returns a clip handle for a BSP inline model (submodel). Index 0 is the world; 1+ are brush entities (doors, platforms, etc.).
- Inputs: `index` — submodel index.
- Outputs/Return: `clipHandle_t` for use in trace/contents calls.
- Side effects: None.
- Calls: Not inferable from this file.

### CM_TempBoxModel
- Signature: `clipHandle_t CM_TempBoxModel(const vec3_t mins, const vec3_t maxs, int capsule)`
- Purpose: Creates a transient axis-aligned box or capsule clip model for use in sweeps.
- Inputs: `mins`/`maxs` — bounding extents; `capsule` — nonzero to use capsule geometry.
- Outputs/Return: Temporary `clipHandle_t`; valid only until next call.
- Side effects: Overwrites a single shared temporary model slot.
- Notes: Not thread-safe by design; caller must not cache the handle across frames.

### CM_BoxTrace
- Signature: `void CM_BoxTrace(trace_t *results, const vec3_t start, const vec3_t end, vec3_t mins, vec3_t maxs, clipHandle_t model, int brushmask, int capsule)`
- Purpose: Sweeps an AABB (or capsule) from `start` to `end` against a clip model, returning the first blocking intersection.
- Inputs: `start`/`end` — sweep endpoints; `mins`/`maxs` — hull extents; `model` — world or submodel handle; `brushmask` — content filter; `capsule` — capsule mode flag.
- Outputs/Return: Fills `results` with fraction, endpos, hit plane, surface flags, contents.
- Side effects: None beyond writing `results`.
- Calls: Not inferable from this file.

### CM_TransformedBoxTrace
- Signature: `void CM_TransformedBoxTrace(trace_t *results, ..., const vec3_t origin, const vec3_t angles, int capsule)`
- Purpose: Same as `CM_BoxTrace` but transforms the trace into the local space of a rotated/translated submodel (e.g., a rotating door).
- Inputs: Same as `CM_BoxTrace` plus `origin`/`angles` for the model's world transform.
- Outputs/Return: Fills `results`.
- Side effects: None.

### CM_PointContents / CM_TransformedPointContents
- Signature: `int CM_PointContents(const vec3_t p, clipHandle_t model)` / `int CM_TransformedPointContents(..., origin, angles)`
- Purpose: Returns the ORed brush contents flags at a world point (e.g., `CONTENTS_WATER`, `CONTENTS_SOLID`).
- Inputs: Point, optional model transform.
- Outputs/Return: Integer contents bitmask.

### CM_ClusterPVS
- Signature: `byte *CM_ClusterPVS(int cluster)`
- Purpose: Returns a pointer to the raw PVS bitset for the given cluster, used to cull potentially invisible areas.
- Outputs/Return: Pointer into the BSP visibility lump; not owned by caller.

### CM_AdjustAreaPortalState / CM_AreasConnected / CM_WriteAreaBits
- Purpose: Manage area portal open/closed state (for doors/gates) and query area connectivity for audio and PVS propagation.

### CM_LerpTag
- Signature: `int CM_LerpTag(orientation_t *tag, clipHandle_t model, int startFrame, int endFrame, float frac, const char *tagName)`
- Purpose: Interpolates a named MD3 attachment tag between two frames for weapon/attachment positioning.
- Outputs/Return: Returns 1 on success, 0 if tag not found.

### CM_MarkFragments
- Signature: `int CM_MarkFragments(int numPoints, const vec3_t *points, const vec3_t projection, int maxPoints, vec3_t pointBuffer, int maxFragments, markFragment_t *fragmentBuffer)`
- Purpose: Clips a projected polygon (decal, bullet mark) against world surfaces to produce renderable fragments.
- Outputs/Return: Number of fragments written to `fragmentBuffer`.

### CM_DrawDebugSurface
- Signature: `void CM_DrawDebugSurface(void (*drawPoly)(int color, int numPoints, float *points))`
- Purpose: Debug visualization hook; calls back into provided `drawPoly` function to render internal patch collision geometry.

## Control Flow Notes
This header is included by server (`sv_*.c`), client (`cl_*.c`), and game VM bridge code. `CM_LoadMap` is called during map load (server init or clientload). Per-frame, `CM_BoxTrace` and `CM_PointContents` are called heavily by physics and visibility code. `CM_ClusterPVS` and `CM_AreasConnected` are used each server frame to build snapshots. `CM_ClearMap` is called on map unload/shutdown.

## External Dependencies
- `qfiles.h` — BSP file format structs (`dheader_t`, `dleaf_t`, etc.) and limits; `vmHeader_t`, model format structs
- `clipHandle_t`, `trace_t`, `vec3_t`, `orientation_t`, `markFragment_t`, `qboolean`, `byte` — all defined in `q_shared.h` or `cm_local.h`, included transitively by consumers
- Implementation bodies defined across `cm_load.c`, `cm_trace.c`, `cm_test.c`, `cm_patch.c`, `cm_tag.c`, `cm_marks.c`
