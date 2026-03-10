# code/qcommon/cm_load.c — Enhanced Analysis

## Architectural Role

This file is the **entry point for all BSP world data initialization** in the qcommon collision subsystem. It deserializes a binary BSP file into the global `clipMap_t cm` singleton, making all geometric data available to collision queries, spatial queries (traces, point-in-area tests), and visibility culling. `CM_LoadMap` is called once per level load by the server (and optionally the client for offline prediction); the loaded data directly feeds the collision trace system, the renderer's PVS-based visibility culling, the botlib navigation system, and the game VM's entity/physics pipeline. It is the bridge from disk to all downstream collision and spatial queries.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/server/sv_init.c`** → `CM_LoadMap` during `SV_SpawnServer` when a new map loads
- **`code/client/cl_cgame.c`** (optional) → `CM_LoadMap` for offline client-side prediction in listen servers
- **`code/game/g_main.c`** → the game VM indirectly depends on loaded `cm` for `trap_Trace`, `trap_PointContents`
- **`code/renderer/tr_world.c`** → uses `cm.leafs`, `cm.clusters`, and PVS data (`CM_ClusterPVS`, called via `refimport` from renderer init)
- **`code/botlib/be_aas_bspq3.c`** → `AAS_LoadBSPFile` parses the BSP entity string from `cm.entityString` to extract bot navigation hints

### Outgoing (what this file depends on)

- **`code/qcommon/files.c`** → `FS_ReadFile`, `FS_FreeFile` to load the raw BSP buffer from disk or `.pk3`
- **`code/qcommon/mem.c`** → `Hunk_Alloc` (h_high allocation for all runtime BSP arrays)
- **`code/qcommon/common.c`** → `Com_Error`, `Com_Memcpy`, `Com_Memset`, error handling and utilities
- **`code/qcommon/cvar.c`** → `Cvar_Get` for `cm_noAreas`, `cm_noCurves`, `cm_playerCurveClip` debug cvars
- **`code/qcommon/cm_patch.c`** → `CM_GeneratePatchCollide` to convert each Bézier surface into collision geometry
- **`code/qcommon/cm_trace.c`** (same compilation unit area) → `CM_FloodAreaConnections` for area-portal connectivity, `CM_InitBoxHull` for synthetic AABB model
- **`code/game/q_math.c`** → `PlaneTypeForNormal`, `LittleLong`, `LittleFloat` for endian conversion and plane classification

## Design Patterns & Rationale

### 1. **Lump-Based BSP Parsing**
Each `CMod_Load*` function follows a stereotyped pattern:
- Cast the lump buffer to the on-disk struct type
- Validate size divisibility
- Calculate element count
- Allocate hunk memory
- Loop: endian-swap each field (LittleLong, LittleFloat)

**Rationale:** BSP files are a legacy binary format designed for fast mmap-load on x86. The lump structure allows incremental parsing with clear dependency ordering. Endian-swapping is mandatory because Q3 always stores little-endian regardless of platform.

### 2. **Extension Slots for Dynamic Geometry**
The collision system pre-allocates extra slots beyond the loaded map:
- `cm.planes`: loaded count + `BOX_PLANES` (12)
- `cm.brushes`: loaded count + `BOX_BRUSHES` (1)
- `cm.brushsides`: loaded count + `BOX_SIDES` (6)
- `cm.leafbrushes`: loaded count + `BOX_BRUSHES` (1)

**Rationale:** Allows `CM_TempBoxModel` to dynamically populate a synthetic brush at init time for transient AABB queries. This avoids allocating per-query; instead, a single shared `box_model` is updated and reused. The trade-off: only one box query can be "live" at a time (callers must not hold the handle across iterations).

### 3. **Identity-Based Reload Prevention**
`CM_LoadMap` caches `last_checksum` and compares it against the computed checksum of the new map. If identical and `clientload` is true, it returns immediately without reloading.

**Rationale:** On a fast client in a listen server, the same map is often loaded twice (once for server, once for client). Skipping the redundant hunk allocation and endian-swapping saves initialization time.

### 4. **Trivial Box AABB Model**
The box hull is a hardcoded 6-sided axially-aligned brush. Rather than allocate per query, it's static, and `CM_TempBoxModel` mutates its plane distances to match the requested AABB.

**Rationale:** Avoids allocation overhead; keeps collision API simple (traces against `clipHandle_t`). The downside: not thread-safe; only one box query at a time.

## Data Flow Through This File

```
BSP file (on disk, little-endian)
    ↓
FS_ReadFile → raw buffer in memory
    ↓
CM_LoadMap (in load order):
  1. CMod_LoadShaders      → cm.shaders[]       (content/surface flags)
  2. CMod_LoadLeafs        → cm.leafs[]         (cluster, area, brush/surface indices)
  3. CMod_LoadLeafBrushes  → cm.leafbrushes[]   (per-leaf brush indices)
  4. CMod_LoadLeafSurfaces → cm.leafsurfaces[]  (per-leaf surface indices)
  5. CMod_LoadPlanes       → cm.planes[]        (plane equations + signbits)
  6. CMod_LoadBrushSides   → cm.brushsides[]    (plane refs + shader flags)
  7. CMod_LoadBrushes      → cm.brushes[]       (side refs, bounds, content)
  8. CMod_LoadSubmodels    → cm.cmodels[]       (inline brush models from BSP)
  9. CMod_LoadNodes        → cm.nodes[]         (BSP tree)
  10. CMod_LoadEntityString → cm.entityString   (entity key/value pairs)
  11. CMod_LoadVisibility   → cm.visibility[]   (PVS cluster data)
  12. CMod_LoadPatches      → cm.surfaces[]     (Bézier patch collision)
    ↓
    [Endian-swap all numeric fields to host byte order]
    ↓
    CM_InitBoxHull()
    CM_FloodAreaConnections()
    ↓
    Global cm is now ready for:
    - Traces (CM_Trace, via cm_trace.c)
    - Point queries (CM_PointContents, via cm_test.c)
    - PVS queries (CM_ClusterPVS, via cm_public.h)
    - Area connectivity (cm.areaPortals[cluster×cluster])
```

**Key dependencies:**
- Planes must be loaded before BrushSides (which reference plane indices)
- BrushSides must be loaded before Brushes (which reference brushside ranges)
- Leafs must be loaded before Nodes (the tree root is in leafs)
- All geometry must be loaded before `CM_InitBoxHull` (which extends the plane/brush arrays)
- Entity string must be available for botlib's entity parsing in `AAS_LoadBSPFile`

## Learning Notes

### Era-Specific Design
- **No streaming**: All BSP geometry is loaded into hunk memory at once. Modern engines stream large open worlds; Q3 assumes the entire level fits in RAM (typical: 1–10 MB per map).
- **PVS-based culling**: The renderer uses *potentially visible set* (cluster-to-cluster visibility) to skip entire regions. This precomputed data is hostile to dynamic geometry; modern engines use real-time occlusion queries or portals.
- **Endian-swapping**: The BSP format stores data in little-endian regardless of platform. On big-endian systems (PowerPC, Alpha), every numeric field must be byteswapped. Modern tools often generate platform-native formats.

### Architectural Insight: Dual-Layer Spatial Structure
Q3 uses **two complementary spatial hierarchies**:
1. **BSP tree** (`cm.nodes`, `cm.leafs`) — stores static brushes; used for collision traces
2. **Area portals** (`cm.areaPortals[]`) — low-res connectivity; used by sound and bot pathing

The area-portal layer is optional; if `cm_noAreas` is set, collision falls back to the full BSP tree. This allows tuning between precision and performance.

### Collision Model Abstraction
The file exports collision data via `clipHandle_t` opaque indices (`CM_InlineModel`, `CM_TempBoxModel`), decoupling the trace API from internal representation. This allows the implementation to change (e.g., switch from brushes to voxels) without breaking callers.

### Visibility for Multiple Subsystems
The loaded PVS data (`cm.visibility`, `cm.clusterBytes`) is consumed by:
- **Renderer** → frustum + PVS culling of surfaces
- **Game VM** → area-aware AI (bot nav, sound propagation)
- **Botlib** → cluster reachability precomputation in `AAS_InitClustering`

A single load operation provides visibility info to all downstream systems.

## Potential Issues

1. **Global State Not Thread-Safe**
   - The global `cm` and `box_model` are not mutex-protected. If `CM_LoadMap` were called from multiple threads, or if `CM_TempBoxModel` were called concurrently with trace operations, undefined behavior would result.
   - Mitigation: Q3 is single-threaded for game logic; only the renderer SMP splits work, and that layer doesn't call `CM_Load*`.

2. **Checksum Cache Assumes Filename Uniqueness**
   - `CM_LoadMap` caches the last checksum and skips reloading if the checksum matches. However, if the same filename contains different content (e.g., user edits `q3dm1.bsp`), the old cached data is used silently.
   - In practice, this is benign: maps are immutable at runtime, and developers would restart the engine to test changes.

3. **Offset-to-Pointer Conversions Are Trust-Based**
   - Functions like `CMod_LoadSubmodels` convert on-disk offsets (e.g., `in->firstBrush`) directly to array indices without bounds checking:
     ```c
     indexes[j] = LittleLong( in->firstBrush ) + j;  // No validation
     ```
   - A malformed BSP could specify an invalid brush index, causing out-of-bounds access later. Mitigation: the BSP compiler (q3map2) is trusted; hand-edited BSPs are risky.

4. **Unvis'd Maps Fill All-Visible**
   - If a map has no visibility data, `CMod_LoadVisibility` fills the visibility array with 0xFF (all clusters visible to all clusters). This is correct for unvis'd maps but can hurt performance on large open-world maps that lack PVS.

---

**Summary:** `cm_load.c` is a foundational infrastructure file that bridges BSP disk format to runtime collision and visibility systems. Its design reflects early-2000s constraints (memory efficiency, compile-time precomputation) and is remarkably stable—the BSP format has not fundamentally changed since Quake I. The file is an excellent reference for understanding the relationship between spatial data structures, endian-aware file I/O, and multi-subsystem initialization in a large C game engine.
