# code/botlib/be_aas_sample.c

## File Purpose
Implements AAS (Area Awareness System) environment sampling for the Quake III bot library. It provides spatial queries against the AAS BSP tree — point-to-area lookup, line tracing, bounding box area enumeration, entity linking/unlinking, and face containment tests used by bot navigation.

## Core Responsibilities
- Map points to AAS area numbers via BSP tree traversal
- Trace a bounding-box sweep through the AAS tree (`AAS_TraceClientBBox`)
- Collect all AAS areas a line segment passes through (`AAS_TraceAreas`)
- Link/unlink game entities into AAS areas for collision queries
- Manage a fixed-size free-list heap of `aas_link_t` nodes
- Test whether a point lies inside a face polygon
- Return area metadata (presence type, cluster, bounding box)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `aas_tracestack_t` | struct | Stack frame for iterative BSP line-trace; stores start/end points, last plane used, and child node number |
| `aas_linkstack_t` | struct | Stack frame for iterative BSP bbox-link traversal; stores child node number only |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `numaaslinks` | `int` | global (file-level) | Running count of free `aas_link_t` slots remaining in the heap |
| `botimport` | `botlib_import_t` (extern) | global | Engine import table; used for printing errors/warnings |

## Key Functions / Methods

### AAS_PresenceTypeBoundingBox
- **Signature:** `void AAS_PresenceTypeBoundingBox(int presencetype, vec3_t mins, vec3_t maxs)`
- **Purpose:** Returns the AABB (mins/maxs) for standing (`PRESENCE_NORMAL`) or crouching (`PRESENCE_CROUCH`) player presence.
- **Inputs:** `presencetype` — one of the `PRESENCE_*` constants.
- **Outputs/Return:** Writes into `mins`/`maxs` (±15, ±15, -24/+32 standing; -24/+8 crouch).
- **Side effects:** Prints `PRT_FATAL` for unknown presence type.
- **Calls:** `botimport.Print`, `VectorCopy`

---

### AAS_InitAASLinkHeap / AAS_FreeAASLinkHeap
- **Signature:** `void AAS_InitAASLinkHeap(void)` / `void AAS_FreeAASLinkHeap(void)`
- **Purpose:** Allocates (or frees) the fixed pool of `aas_link_t` nodes used for entity-area linking. Reads `max_aaslinks` cvar (default 6144) unless running in BSPC mode.
- **Side effects:** Allocates hunk memory; initialises doubly-linked free list; sets `aasworld.freelinks`, `aasworld.linkheap`, `numaaslinks`.
- **Calls:** `GetHunkMemory`, `LibVarValue`, `FreeMemory`

---

### AAS_AllocAASLink / AAS_DeAllocAASLink
- **Signature:** `aas_link_t *AAS_AllocAASLink(void)` / `void AAS_DeAllocAASLink(aas_link_t *link)`
- **Purpose:** O(1) allocation/deallocation from the pre-allocated link heap free list.
- **Side effects:** Mutates `aasworld.freelinks`, decrements/increments `numaaslinks`.
- **Notes:** Returns `NULL` and prints fatal error when the heap is exhausted.

---

### AAS_PointAreaNum
- **Signature:** `int AAS_PointAreaNum(vec3_t point)`
- **Purpose:** Walks the AAS BSP tree from root to leaf to find which area number contains `point`. Returns 0 for solid or unloaded AAS.
- **Inputs:** World-space point.
- **Outputs/Return:** AAS area index (positive), or 0 if in solid/not loaded.
- **Calls:** `DotProduct`, `botimport.Print`

---

### AAS_PointReachabilityAreaIndex
- **Signature:** `int AAS_PointReachabilityAreaIndex(vec3_t origin)`
- **Purpose:** Converts a world point into a flat reachability-area index across all clusters (used for global routing tables). Passing `NULL` returns the total count.
- **Calls:** `AAS_PointAreaNum`, `AAS_AreaReachability`

---

### AAS_TraceClientBBox
- **Signature:** `aas_trace_t AAS_TraceClientBBox(vec3_t start, vec3_t end, int presencetype, int passent)`
- **Purpose:** Iterative BSP trace of a swept bounding box from `start` to `end`. Returns hit info including fraction, end position, area, and plane. Also tests entity collision per area when `passent >= 0`.
- **Inputs:** Start/end points, presence type (determines bbox), entity number to ignore.
- **Outputs/Return:** `aas_trace_t` with `startsolid`, `fraction`, `endpos`, `area`, `planenum`.
- **Side effects:** None (read-only AAS data); may call entity collision via `AAS_AreaEntityCollision`.
- **Calls:** `AAS_AreaEntityCollision`, `DotProduct`, `VectorCopy`, `VectorSubtract`, `VectorLength`, `VectorNormalize`, `VectorMA`, `Com_Memset`, `botimport.Print`
- **Notes:** Uses a 127-deep explicit stack (`tracestack`). Axial-plane fast-path is commented out with a FIXME due to incorrect results.

---

### AAS_TraceAreas
- **Signature:** `int AAS_TraceAreas(vec3_t start, vec3_t end, int *areas, vec3_t *points, int maxareas)`
- **Purpose:** Collects all AAS area numbers (and optionally entry points) the line segment passes through, up to `maxareas`.
- **Outputs/Return:** Count of areas found; fills `areas[]` and optional `points[]`.
- **Calls:** `DotProduct`, `VectorCopy`, `botimport.Print`

---

### AAS_AASLinkEntity
- **Signature:** `aas_link_t *AAS_AASLinkEntity(vec3_t absmins, vec3_t absmaxs, int entnum)`
- **Purpose:** Iterative BSP descent using `AAS_BoxOnPlaneSide2` to find all AAS areas overlapping the given AABB; allocates `aas_link_t` nodes linking the entity into each overlapping area.
- **Outputs/Return:** Head of the per-entity area-link list.
- **Calls:** `AAS_AllocAASLink`, `AAS_BoxOnPlaneSide2`, `botimport.Print`

---

### AAS_UnlinkFromAreas
- **Signature:** `void AAS_UnlinkFromAreas(aas_link_t *areas)`
- **Purpose:** Removes an entity from all areas it was linked into, then deallocates those link nodes.
- **Calls:** `AAS_DeAllocAASLink`

---

### AAS_InsideFace / AAS_PointInsideFace
- **Purpose:** Test whether a 3D point lies inside a convex face polygon using edge separation normals. `AAS_InsideFace` takes an external plane normal; `AAS_PointInsideFace` looks it up from the face's stored plane.
- **Notes:** Uses the `AAS_OrthogonalToVectors` macro (cross product) for separation-plane normals.

---

### AAS_AreaGroundFace / AAS_TraceEndFace
- **Purpose:** Helper queries — `AAS_AreaGroundFace` returns the `FACE_GROUND`-flagged face beneath a point in an area; `AAS_TraceEndFace` finds the face that a trace endpoint landed on.

---

### AAS_AreaInfo
- **Signature:** `int AAS_AreaInfo(int areanum, aas_areainfo_t *info)`
- **Purpose:** Fills an `aas_areainfo_t` with cluster, contents, flags, presence type, and bounds for a given area.

## Control Flow Notes
This file has no frame/tick callback of its own. It is called during bot thinking frames: `AAS_PointAreaNum` and `AAS_TraceClientBBox` are the primary spatial query entry points invoked by the movement and routing subsystems. `AAS_InitAASLinkHeap` and `AAS_InitAASLinkedEntities` are called at AAS world load time; their `Free*` counterparts at unload. Entity linking (`AAS_AASLinkEntity`) is called each server frame when entities move.

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `DotProduct`, `VectorCopy`, `VectorSubtract`, `VectorMA`, `VectorNormalize`, `Com_Memset`, `qboolean`
- `l_memory.h` — `GetHunkMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_libvar.h` (non-BSPC) — `LibVarValue` to read `max_aaslinks` cvar
- `be_aas_def.h` — `aasworld` global (type `aas_world_t`), `aas_link_t`, `aas_node_t`, `aas_plane_t`, `aas_face_t`, `aas_edge_t`, `aas_area_t`, `aas_areasettings_t`
- `be_aas_funcs.h` — `AAS_EntityCollision`, `AAS_AreaReachability` (defined elsewhere)
- `be_interface.h` — `bot_developer` flag (defined elsewhere)
- `botlib_import_t botimport` — engine printing/import functions (defined in `be_interface.c`)
