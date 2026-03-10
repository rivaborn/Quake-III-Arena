# code/botlib/be_aas_sample.h — Enhanced Analysis

## Architectural Role

This header forms the **spatial query API layer** of botlib's Area Awareness System. It decouples high-level bot movement/navigation (which call these functions) from internal AAS geometry traversal and the global `aasworld` singleton. The file is split into two tiers: internal geometry helpers (guarded by `AASINTERN`, for use only by `be_aas_sample.c` itself) and public spatial queries (exposed via `botlib_export_t` to the server's game VM, which drives bot ticks). Together, these functions answer all spatial questions the bot AI stack needs: "Where am I?" (point-to-area), "Can I move there?" (trace queries), "What areas do I touch?" (bbox enumeration).

## Key Cross-References

### Incoming (who calls this file's functions)
- **`code/botlib/be_aas_reach.c`**: Uses `AAS_PointAreaNum`, `AAS_AreaCluster`, `AAS_TraceClientBBox` to compute reachability between areas (jump arcs, water transitions, etc.)
- **`code/botlib/be_aas_route.c`**: Calls `AAS_AreaCluster` and routing predicates to find inter-cluster paths
- **`code/botlib/be_aas_move.c`**: Calls `AAS_TraceClientBBox`, `AAS_PresenceTypeBoundingBox` for movement prediction and collision validation
- **`code/botlib/be_aas_entity.c`**: Uses entity linking (`AASINTERN` functions) to maintain spatial index of dynamic entities
- **Server game VM** (`code/game/g_bot.c`, `ai_dmq3.c`): Calls public API functions (point area, trace, bbox queries) indirectly through botlib's higher-level functions exposed via `botlib_export_t` vtable
- **`code/botlib/be_aas_debug.c`**: Calls debug visualizers that internally use `AAS_PointAreaNum`, etc.

### Outgoing (what this file depends on)
- **`code/botlib/be_aas_def.h`**: Defines `aas_face_t`, `aas_plane_t`, `aas_link_t`, `aas_trace_t`, `aas_areainfo_t` structures consumed by all declarations
- **Global `aasworld` singleton** (defined in `be_aas_main.c`): All functions read the loaded AAS geometry and spatial hierarchy (areas, faces, planes, clusters)
- **`code/qcommon/cm_*.c` collision model**: Traces are validated against the engine's BSP collision geometry (inferred from `AAS_TraceClientBBox` semantics)

## Design Patterns & Rationale

### 1. **Conditional API Exposure (AASINTERN Guard)**
Internal helpers (`AAS_AASLinkEntity`, `AAS_AreaGroundFace`, `AAS_PlaneFromNum`) are hidden from external callers by `#ifdef AASINTERN`. This enforces **API boundaries**: only `be_aas_sample.c` and a few internal files directly manipulate the geometry hierarchy. Higher-level code must route through public queries, ensuring consistency.

### 2. **Presence-Type Parameterization**
Functions like `AAS_PresenceTypeBoundingBox`, `AAS_TraceClientBBox`, and `AAS_LinkEntityClientBBox` accept an `int presencetype` parameter instead of fixed dimensions. This is **idiomatic Q3A**: bots and players have distinct postures (standing, crouching, flying) with different collision volumes. A single query function adapts to all postures, avoiding code duplication across bot AI.

### 3. **Entity Linking as Spatial Index**
`AAS_AASLinkEntity` and `AAS_LinkEntityClientBBox` build a per-area linked list of overlapping entities (`aas_link_t` chains). This avoids expensive per-query spatial tree traversals—instead, entities are pre-linked into their areas, and querying is O(1) per area. The `AASINTERN` link management functions implement a **lazy spatial index** updated on demand.

### 4. **Out-Parameter Return for Complex Results**
`AAS_TraceAreas` and `AAS_BBoxAreas` return counts but fill caller-supplied arrays (`int *areas`, `vec3_t *points`). This avoids dynamic allocation and aligns with Q3A's hunk-buffer discipline. The `maxareas` cap prevents buffer overruns.

## Data Flow Through This File

### Point-to-Area Query Path
1. Bot queries `AAS_PointAreaNum(bot_position)` → finds which AAS area the bot occupies
2. Used by higher-level `AAS_AreaCluster`, `AAS_AreaPresenceType` to determine bot's current cluster and valid movement modes
3. Result feeds into routing and goal evaluation

### Trace & Movement Validation Path
1. Reachability checker calls `AAS_TraceClientBBox(from, to, presencetype, passent)` → simulates a client-sized sweep
2. Returns `aas_trace_t` with fraction, end position, and hit face/area
3. Used by `be_aas_reach.c` to classify whether a jump/walk is reachable and how much clearance exists
4. Movement predictor (`be_aas_move.c`) calls same trace to validate predicted paths

### BBox Enumeration Path
1. `AAS_BBoxAreas(mins, maxs, areas[], max)` → collects all AAS areas overlapping a bounding box
2. Called when a dynamic entity (item, projectile) spawns to link it into the spatial index
3. `AAS_UnlinkFromAreas` removes the entity when it despawns

## Learning Notes

### Idiomatic Q3A Patterns
- **Presence types** (`PRESENCE_NORMAL`, `PRESENCE_CROUCH`, `PRESENCE_SWIM`) are Q3A-specific and absent in modern engines (which use collision shapes/layers)
- **Entity linking** is a hand-rolled spatial index; modern engines use spatial hash grids or octrees
- **Out-parameters** reflect early-2000s C conventions predating statics constructors or RAII

### Key Insight: Separation of Concerns
The `AASINTERN` vs public split reveals **deep modularity**:
- Internal layer: geometry primitives (faces, planes, point-in-face tests)
- Public layer: abstract spatial queries (area, cluster, trace)
- Higher layers (`reach`, `route`, `move`) know nothing of AAS geometry details

This is analogous to **renderer front-end/back-end** split in `code/renderer/`: front-end (scene collection) is public; back-end (GL command dispatch) is internal.

### Reachability & Movement Symbiosis
`AAS_TraceClientBBox` is called by **both** reachability analysis (can I reach area B from A?) and movement simulation (will this velocity vector hit walls?). This dual use ensures bot pathfinding and movement are mutually consistent—critical for Q3A bot believability.

## Potential Issues

1. **No bounds checking visible**: `AAS_TraceAreas` and `AAS_BBoxAreas` assume caller respects `maxareas`; buffer overflow is possible if caller miscalculates capacity.
2. **Presence type coupling**: All high-level routing assumes presence types are known at query time. If a bot's posture can change mid-movement, stale traces could cause clipping.
3. **Entity linking heap exhaustion**: `AAS_InitAASLinkHeap` allocates a fixed pool; heavy dynamic entity loading (many items/projectiles) could exhaust it without explicit diagnostics.
