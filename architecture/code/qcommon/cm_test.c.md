# code/qcommon/cm_test.c

## File Purpose
Implements point/area spatial queries and area connectivity (PVS/portals) for Quake III Arena's collision map system. It provides BSP tree traversal to locate leafs and brush contents at a point, as well as flood-fill logic for area portal connectivity used in entity culling.

## Core Responsibilities
- Traverse the BSP tree to find which leaf a point occupies
- Query content flags (solid, water, etc.) at a world-space point
- Enumerate all leafs or brushes touching an AABB
- Expose PVS (Potentially Visible Set) cluster data per cluster index
- Maintain and recompute area portal flood-connectivity
- Provide a bit-vector of areas reachable from a given area for snapshot culling

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `leafList_t` | struct (defined in cm_local.h) | Accumulator passed through recursive leaf enumeration; holds result list, bounds, overflow flag, and a function pointer for storage callback |
| `cNode_t` | struct | BSP tree node with plane pointer and two child indices |
| `cLeaf_t` | struct | BSP leaf referencing brush and surface index ranges |
| `cbrush_t` | struct | Convex brush with sides, bounds, contents flags, and a checkcount for deduplication |
| `cArea_t` | struct | Area with flood number and validity stamp for connectivity tracking |
| `sphere_t` / `traceWork_t` | structs | Defined here (via cm_local.h); used by cm_trace.c, not directly by this file |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cm` | `clipMap_t` | global (extern) | The loaded collision map; all BSP, brush, leaf, area, and visibility data |
| `c_pointcontents` | `int` | global (extern) | Performance counter incremented on each point-leaf lookup |
| `cm_noAreas` | `cvar_t *` | global (extern) | CVar; when set, bypasses area connectivity checks |

## Key Functions / Methods

### CM_PointLeafnum_r
- **Signature:** `int CM_PointLeafnum_r( const vec3_t p, int num )`
- **Purpose:** Recursively (iteratively) walks the BSP tree from node `num` to find the leaf index containing point `p`.
- **Inputs:** World-space point `p`; starting node index `num` (0 = root).
- **Outputs/Return:** Leaf index (non-negative).
- **Side effects:** Increments `c_pointcontents`.
- **Calls:** `DotProduct` (macro).
- **Notes:** Uses axial-plane fast path (`plane->type < 3`). Terminates when `num < 0`; leaf index = `-1 - num`.

### CM_PointLeafnum
- **Signature:** `int CM_PointLeafnum( const vec3_t p )`
- **Purpose:** Public entry point; guards against unloaded map, then calls `CM_PointLeafnum_r` from root.
- **Inputs:** World-space point.
- **Outputs/Return:** Leaf index, or 0 if map not loaded.
- **Side effects:** Via `CM_PointLeafnum_r`.
- **Calls:** `CM_PointLeafnum_r`.

### CM_StoreLeafs
- **Signature:** `void CM_StoreLeafs( leafList_t *ll, int nodenum )`
- **Purpose:** Callback used by `CM_BoxLeafnums_r` to append a leaf index to the result list.
- **Side effects:** Sets `ll->lastLeaf` if the leaf has a valid cluster; sets `ll->overflowed` on capacity exceeded.

### CM_StoreBrushes
- **Signature:** `void CM_StoreBrushes( leafList_t *ll, int nodenum )`
- **Purpose:** Callback variant that appends unique `cbrush_t*` pointers (AABB-filtered) into the list instead of leaf indices.
- **Side effects:** Uses `b->checkcount` / `cm.checkcount` to skip duplicate brushes across leaves.

### CM_BoxLeafnums_r
- **Signature:** `void CM_BoxLeafnums_r( leafList_t *ll, int nodenum )`
- **Purpose:** Iterative/recursive BSP traversal collecting all leafs whose subtrees overlap `ll->bounds`.
- **Calls:** `BoxOnPlaneSide`, `ll->storeLeafs` (function pointer), itself recursively for straddle case.

### CM_BoxLeafnums
- **Signature:** `int CM_BoxLeafnums( const vec3_t mins, const vec3_t maxs, int *list, int listsize, int *lastLeaf )`
- **Purpose:** Public API to collect all leaf indices touching the given AABB.
- **Calls:** `CM_BoxLeafnums_r`. Increments `cm.checkcount`.
- **Outputs/Return:** Count of leafs found; fills `list`; sets `*lastLeaf`.

### CM_BoxBrushes
- **Signature:** `int CM_BoxBrushes( const vec3_t mins, const vec3_t maxs, cbrush_t **list, int listsize )`
- **Purpose:** Collects all unique brushes touching the AABB, using `CM_StoreBrushes` as the callback.
- **Outputs/Return:** Count of brushes found.

### CM_PointContents
- **Signature:** `int CM_PointContents( const vec3_t p, clipHandle_t model )`
- **Purpose:** Returns the ORed content flags of all brushes containing point `p` in the given model (or world).
- **Inputs:** Point; optional submodel handle (0 = world).
- **Calls:** `CM_ClipHandleToModel`, `CM_PointLeafnum_r`, `DotProduct`.
- **Notes:** A point is inside a brush only if it is on the negative side of all brush planes. The `> dist` vs `>= dist` comparison has a `FIXME` comment indicating a known edge-case.

### CM_TransformedPointContents
- **Signature:** `int CM_TransformedPointContents( const vec3_t p, clipHandle_t model, const vec3_t origin, const vec3_t angles )`
- **Purpose:** Wraps `CM_PointContents` after transforming the point into the local frame of a rotated/translated entity.
- **Calls:** `VectorSubtract`, `AngleVectors`, `DotProduct`, `CM_PointContents`.
- **Notes:** Rotation is skipped for `BOX_MODEL_HANDLE` or zero angles.

### CM_ClusterPVS
- **Signature:** `byte *CM_ClusterPVS( int cluster )`
- **Purpose:** Returns a pointer into the raw PVS visibility data for the given cluster.
- **Outputs/Return:** Pointer to a bitmask row; returns `cm.visibility` (all-visible fallback) if cluster is invalid or map is not vised.

### CM_FloodArea_r
- **Signature:** `void CM_FloodArea_r( int areaNum, int floodnum )`
- **Purpose:** Recursive flood-fill that tags all areas reachable through open portals with the same `floodnum`.
- **Side effects:** Mutates `cm.areas[].floodnum` and `cm.areas[].floodvalid`. Calls `Com_Error` if a re-flood cycle is detected.

### CM_FloodAreaConnections
- **Signature:** `void CM_FloodAreaConnections( void )`
- **Purpose:** Resets flood validity and re-floods all areas to recompute connectivity after portal state changes.
- **Calls:** `CM_FloodArea_r` per unvisited area.

### CM_AdjustAreaPortalState
- **Signature:** `void CM_AdjustAreaPortalState( int area1, int area2, qboolean open )`
- **Purpose:** Increments or decrements the reference count of a portal between two areas, then triggers a full flood recompute.
- **Side effects:** Modifies `cm.areaPortals[]`; calls `CM_FloodAreaConnections`.
- **Notes:** Reference-counted to support multiple simultaneous openers. Negative count triggers `Com_Error`.

### CM_AreasConnected
- **Signature:** `qboolean CM_AreasConnected( int area1, int area2 )`
- **Purpose:** Returns true if both areas share the same flood number (i.e., are connected through open portals).
- **Notes:** Bypassed entirely if `cm_noAreas` CVar is set.

### CM_WriteAreaBits
- **Signature:** `int CM_WriteAreaBits( byte *buffer, int area )`
- **Purpose:** ORs a bitmask into `buffer` marking all areas in the same flood as `area`; used for snapshot visibility culling.
- **Outputs/Return:** Number of bytes in the bitmask (`(numAreas+7)/8`).
- **Notes:** `area == -1` or `cm_noAreas` causes all bits to be set (debug/fallback path).

## Control Flow Notes
This file is part of the **collision map subsystem** (`cm_*.c`). It is called during:
- **Per-frame / per-entity update**: `CM_PointContents` and `CM_TransformedPointContents` are called by game/server logic to determine what medium an entity is standing in.
- **Snapshot generation**: `CM_WriteAreaBits` and `CM_AreasConnected` are called by the server snapshot system to cull non-visible entities.
- **Portal state change events**: `CM_AdjustAreaPortalState` is called by server game logic when doors open/close, triggering `CM_FloodAreaConnections`.
- No rendering or per-frame tick ownership; purely query/state-update.

## External Dependencies
- `cm_local.h` → pulls in `q_shared.h`, `qcommon.h`, `cm_polylib.h`
- **Defined elsewhere:** `cm` (clipMap_t global, loaded by `cm_load.c`), `CM_ClipHandleToModel` (`cm_load.c`), `BoxOnPlaneSide` (`q_shared.c`/math), `AngleVectors`, `DotProduct`, `VectorCopy`, `VectorSubtract` (math macros/functions), `Com_Error`, `Com_Memset` (qcommon), `cm_noAreas` CVar (registered in `cm_main.c` or similar).
