# code/qcommon/cm_load.c

## File Purpose
Loads and parses a Quake III BSP map file into the collision map (`clipMap_t cm`) used by the collision detection system. It deserializes all BSP lumps into runtime structures and initializes the box hull and area flood connectivity after loading.

## Core Responsibilities
- Read and validate a BSP file from disk (or BSPC tool path)
- Deserialize each BSP lump (shaders, planes, nodes, leafs, brushes, brush sides, submodels, visibility, patches, entity string) into hunk-allocated runtime structures
- Endian-swap all numeric fields from little-endian BSP format to host byte order
- Compute and expose a checksum for map integrity verification
- Set up the synthetic box hull for AABB-as-brush-model queries
- Provide accessor functions for cluster/area/entity string data

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `clipMap_t` | struct (defined in `cm_local.h`) | Global collision map holding all loaded BSP data arrays |
| `cmodel_t` | struct | Per-submodel bounds and leaf reference |
| `cbrush_t` | struct | Convex brush with sides, content flags, and AABB bounds |
| `cbrushside_t` | struct | One face of a brush, referencing a plane and shader |
| `cNode_t` | struct | BSP tree node with plane and two children |
| `cLeaf_t` | struct | BSP leaf with cluster, area, and brush/surface index ranges |
| `cPatch_t` | struct | Patch surface with generated collision data |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cm` | `clipMap_t` | global | Single active collision map; entire BSP runtime state |
| `c_pointcontents` | `int` | global | Performance counter for point-in-contents queries |
| `c_traces`, `c_brush_traces`, `c_patch_traces` | `int` | global | Performance counters for trace types |
| `cmod_base` | `byte *` | global | Base pointer into the raw BSP file buffer during loading |
| `cm_noAreas`, `cm_noCurves`, `cm_playerCurveClip` | `cvar_t *` | global | Cheat/debug cvars controlling collision behavior |
| `box_model` | `cmodel_t` | global | Synthetic model used for transient AABB collision queries |
| `box_planes` | `cplane_t *` | global | Points into `cm.planes` extension for box hull planes |
| `box_brush` | `cbrush_t *` | global | Points into `cm.brushes` extension for box hull brush |
| `last_checksum` | `unsigned` (static in `CM_LoadMap`) | static | Caches last map checksum to skip reloading same map |

## Key Functions / Methods

### CM_LoadMap
- **Signature:** `void CM_LoadMap( const char *name, qboolean clientload, int *checksum )`
- **Purpose:** Main entry point — reads the BSP file, calls all `CMod_Load*` functions in dependency order, then initializes box hull and area connections.
- **Inputs:** `name` = BSP path; `clientload` = skip re-init if same map; `checksum` = out-param for integrity value.
- **Outputs/Return:** void; populates global `cm`; writes `*checksum`.
- **Side effects:** Zeroes and rebuilds entire `cm` global; allocates hunk memory; frees file buffer via `FS_FreeFile`; registers cvars.
- **Calls:** `FS_ReadFile`, `CMod_LoadShaders`, `CMod_LoadLeafs`, `CMod_LoadLeafBrushes`, `CMod_LoadLeafSurfaces`, `CMod_LoadPlanes`, `CMod_LoadBrushSides`, `CMod_LoadBrushes`, `CMod_LoadSubmodels`, `CMod_LoadNodes`, `CMod_LoadEntityString`, `CMod_LoadVisibility`, `CMod_LoadPatches`, `CM_InitBoxHull`, `CM_FloodAreaConnections`, `CM_ClearLevelPatches`.
- **Notes:** If `name` matches `cm.name` and `clientload` is true, returns immediately with cached checksum. Empty name initializes a trivial 1-leaf map.

### CM_InitBoxHull
- **Signature:** `void CM_InitBoxHull( void )`
- **Purpose:** Appends 12 planes and 6 brush sides beyond the loaded map data to represent a dynamic AABB, allowing bounding boxes to be treated as inline BSP models.
- **Inputs:** None (uses global `cm` counts as base offsets).
- **Outputs/Return:** void; sets `box_planes`, `box_brush`, `box_model`.
- **Side effects:** Writes into the extra slots pre-allocated in `cm.planes`, `cm.brushsides`, `cm.leafbrushes`.
- **Calls:** `SetPlaneSignbits`, `VectorClear`.

### CM_TempBoxModel
- **Signature:** `clipHandle_t CM_TempBoxModel( const vec3_t mins, const vec3_t maxs, int capsule )`
- **Purpose:** Updates the 12 box hull plane distances to match a given AABB; returns the `BOX_MODEL_HANDLE` (or `CAPSULE_MODEL_HANDLE` for capsule queries).
- **Side effects:** Mutates `box_planes[0..11].dist` and `box_brush->bounds`; not thread-safe (single shared box model).
- **Notes:** Only one temporary box exists at a time — callers must not hold the handle across frames.

### CMod_LoadPatches
- **Signature:** `void CMod_LoadPatches( lump_t *surfs, lump_t *verts )`
- **Purpose:** Iterates BSP surfaces, skips non-patch types, and generates collision data for each patch via `CM_GeneratePatchCollide`.
- **Calls:** `CM_GeneratePatchCollide`.
- **Notes:** Non-patch entries in `cm.surfaces[]` are left NULL.

### CMod_LoadVisibility
- **Signature:** `void CMod_LoadVisibility( lump_t *l )`
- **Purpose:** Loads PVS cluster data; if lump is empty, fills all-visible (0xFF) visibility for unvis'd maps.
- **Side effects:** Sets `cm.vised`, `cm.numClusters`, `cm.clusterBytes`, `cm.visibility`.

### Notes (trivial helpers)
- `CMod_LoadShaders`, `CMod_LoadNodes`, `CMod_LoadBrushes`, `CMod_LoadLeafs`, `CMod_LoadPlanes`, `CMod_LoadLeafBrushes`, `CMod_LoadLeafSurfaces`, `CMod_LoadBrushSides`, `CMod_LoadSubmodels`, `CMod_LoadEntityString` — each reads one BSP lump, endian-swaps fields, and hunk-allocates the corresponding `cm.*` array.
- `CM_BoundBrush` — derives AABB from the first 6 brush sides' plane distances.
- `CM_Checksum` / `CM_LumpChecksum` — compute a combined CRC over 11 lumps for map integrity.
- `CM_ClipHandleToModel`, `CM_InlineModel`, `CM_ModelBounds`, `CM_NumClusters`, `CM_NumInlineModels`, `CM_EntityString`, `CM_LeafCluster`, `CM_LeafArea` — simple index-to-pointer accessors with bounds checking.

## Control Flow Notes
`CM_LoadMap` is called once at level load by the server (and optionally the client). It executes entirely before any game or render frame begins. There is no per-frame work in this file; all functions are init-time or query-time. `CM_ClearMap` is called on level shutdown.

## External Dependencies
- `cm_local.h` → `q_shared.h`, `qcommon.h`, `cm_polylib.h` — shared types and engine utilities
- `bspc/l_qfiles.h` — BSPC tool file abstraction (included only when `BSPC` is defined)
- **Defined elsewhere:** `Hunk_Alloc`, `Com_Memcpy`, `Com_Memset`, `Com_Error`, `Com_DPrintf`, `Com_BlockChecksum`, `FS_ReadFile`, `FS_FreeFile`, `Cvar_Get`, `LittleLong`, `LittleFloat`, `PlaneTypeForNormal`, `SetPlaneSignbits`, `VectorCopy`, `VectorClear`, `Q_strncpyz`, `CM_GeneratePatchCollide`, `CM_ClearLevelPatches`, `CM_FloodAreaConnections`
