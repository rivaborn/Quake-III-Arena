# code/server/sv_world.c

## File Purpose
Implements server-side spatial partitioning and world query operations for Quake III Arena. It maintains an axis-aligned BSP sector tree for fast entity lookups and provides collision tracing, area queries, and point-contents testing against both world geometry and game entities.

## Core Responsibilities
- Build and manage a uniform spatial subdivision tree (`worldSector_t`) for entity bucketing
- Link/unlink game entities into the sector tree when they move or change bounds
- Compute and cache PVS cluster memberships and area numbers per entity on link
- Query all entities whose AABBs overlap a given region (`SV_AreaEntities`)
- Perform swept-box traces through the world and all solid entities (`SV_Trace`)
- Clip a movement against a single specific entity (`SV_ClipToEntity`)
- Return combined content flags at a world point across all overlapping entities (`SV_PointContents`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `worldSector_t` | struct | Node in the uniform binary space partition; stores split axis/distance, two children, and a linked list of entities at this node |
| `areaParms_t` | struct | Scratch parameters passed recursively through `SV_AreaEntities_r`; holds query bounds, output list, and counts |
| `moveclip_t` | struct | Aggregates all state for a single `SV_Trace` call: moving object bounds, trace result, pass-entity exclusion, content mask |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `sv_worldSectors` | `worldSector_t[AREA_NODES]` | global (file) | Fixed-size pool of sector tree nodes; max 64 nodes |
| `sv_numworldSectors` | `int` | global (file) | Allocation cursor into `sv_worldSectors` |

## Key Functions / Methods

### SV_ClipHandleForEntity
- **Signature:** `clipHandle_t SV_ClipHandleForEntity( const sharedEntity_t *ent )`
- **Purpose:** Returns a CM collision handle for any entity — either an inline BSP model handle or a temporary box/capsule model.
- **Inputs:** Pointer to a shared game entity.
- **Outputs/Return:** `clipHandle_t` usable with CM trace functions.
- **Side effects:** May allocate a temporary box model in the collision system.
- **Calls:** `CM_InlineModel`, `CM_TempBoxModel`
- **Notes:** Non-bmodel entities always use axis-aligned boxes; `SVF_CAPSULE` flag selects capsule shape.

### SV_ClearWorld
- **Signature:** `void SV_ClearWorld( void )`
- **Purpose:** Resets the sector tree and rebuilds it to cover the loaded map's bounds.
- **Inputs:** None (reads world model 0 via CM).
- **Outputs/Return:** None.
- **Side effects:** Zeroes `sv_worldSectors`, resets `sv_numworldSectors`, recursively allocates nodes via `SV_CreateworldSector`.
- **Calls:** `CM_InlineModel`, `CM_ModelBounds`, `SV_CreateworldSector`

### SV_CreateworldSector
- **Signature:** `worldSector_t *SV_CreateworldSector( int depth, vec3_t mins, vec3_t maxs )`
- **Purpose:** Recursively builds a uniform binary tree splitting on the longest horizontal axis up to `AREA_DEPTH` (4) levels, yielding at most 16 leaves within the 64-node pool.
- **Inputs:** Current recursion depth, bounding box of this node's region.
- **Outputs/Return:** Pointer to the allocated `worldSector_t` node.
- **Side effects:** Increments `sv_numworldSectors`; writes into `sv_worldSectors[]`.
- **Calls:** Itself (recursive), `VectorSubtract`, `VectorCopy`
- **Notes:** Only X and Y axes are considered for splitting; Z is ignored.

### SV_LinkEntity
- **Signature:** `void SV_LinkEntity( sharedEntity_t *gEnt )`
- **Purpose:** Inserts an entity into the sector tree, encodes its solid size into `entityState_t.solid` for client prediction, and computes PVS cluster and area membership.
- **Inputs:** Pointer to the game entity being linked.
- **Outputs/Return:** None.
- **Side effects:** Writes `gEnt->s.solid`, `gEnt->r.absmin/absmax`, `gEnt->r.linked`, `gEnt->r.linkcount`; updates `svEntity_t` cluster/area fields; modifies `worldSector_t.entities` linked list.
- **Calls:** `SV_SvEntityForGentity`, `SV_UnlinkEntity`, `RadiusFromBounds`, `CM_BoxLeafnums`, `CM_LeafArea`, `CM_LeafCluster`
- **Notes:** Expands AABB by 1 unit on all sides as an epsilon guard. Entities outside the map (zero leafs) are silently not linked.

### SV_UnlinkEntity
- **Signature:** `void SV_UnlinkEntity( sharedEntity_t *gEnt )`
- **Purpose:** Removes an entity from whichever sector it currently occupies.
- **Inputs:** Pointer to the game entity.
- **Side effects:** Sets `gEnt->r.linked = qfalse`, nulls `svEntity_t.worldSector`, splices entity out of sector's linked list.
- **Calls:** `SV_SvEntityForGentity`

### SV_AreaEntities / SV_AreaEntities_r
- **Signature:** `int SV_AreaEntities( const vec3_t mins, const vec3_t maxs, int *entityList, int maxcount )`
- **Purpose:** Collects entity numbers whose stored AABBs overlap the query box using the sector tree for early rejection.
- **Inputs:** Query AABB, output array pointer, array capacity.
- **Outputs/Return:** Number of entities written into `entityList`.
- **Side effects:** None (read-only traversal).
- **Calls:** `SV_AreaEntities_r` (recursive), `SV_GEntityForSvEntity`

### SV_Trace
- **Signature:** `void SV_Trace( trace_t *results, const vec3_t start, vec3_t mins, vec3_t maxs, const vec3_t end, int passEntityNum, int contentmask, int capsule )`
- **Purpose:** Full swept-box trace: first clips against the static world BSP, then clips against all entities whose bounding boxes intersect the move envelope, preserving the closest hit.
- **Inputs:** Start/end points, box half-extents, entity to ignore, content filter, capsule flag.
- **Outputs/Return:** Fills `*results` with closest trace hit data.
- **Side effects:** None beyond writing `*results`.
- **Calls:** `CM_BoxTrace`, `SV_ClipMoveToEntities`

### SV_ClipMoveToEntities
- **Signature:** `void SV_ClipMoveToEntities( moveclip_t *clip )`
- **Purpose:** Iterates entities in the move's bounding region and clips the movement against each, updating `clip->trace` to the closest hit; skips pass-entity and owner relationships.
- **Side effects:** Modifies `clip->trace`.
- **Calls:** `SV_AreaEntities`, `SV_GentityNum`, `SV_ClipHandleForEntity`, `CM_TransformedBoxTrace`

### SV_PointContents
- **Signature:** `int SV_PointContents( const vec3_t p, int passEntityNum )`
- **Purpose:** Returns OR-combined content flags from the world BSP and all entities overlapping point `p`.
- **Outputs/Return:** Integer bitmask of `CONTENTS_*` flags.
- **Calls:** `CM_PointContents`, `SV_AreaEntities`, `SV_GentityNum`, `SV_ClipHandleForEntity`, `CM_TransformedPointContents`

## Control Flow Notes
`SV_ClearWorld` is called once during `SV_SpawnServer` (map load) after the CM world is loaded, before any entities are spawned. `SV_LinkEntity`/`SV_UnlinkEntity` are called per-entity throughout the game frame whenever an entity moves or changes shape. `SV_Trace`, `SV_AreaEntities`, and `SV_PointContents` are called on-demand by the game VM and server systems each frame.

## External Dependencies
- **`server.h`** → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `CM_InlineModel`, `CM_ModelBounds`, `CM_BoxLeafnums`, `CM_LeafArea`, `CM_LeafCluster`, `CM_BoxTrace`, `CM_TransformedBoxTrace`, `CM_TransformedPointContents`, `CM_PointContents`, `CM_TempBoxModel`; `SV_SvEntityForGentity`, `SV_GEntityForSvEntity`, `SV_GentityNum`; `RadiusFromBounds`; globals `sv`, `svs`
