# code/botlib/be_aas_def.h — Enhanced Analysis

## Architectural Role

This header is the **schema spine** of the entire botlib AAS subsystem, serving as the master index into all precomputed navigation data. The monolithic `aas_t` structure couples all AAS subsystems (cluster, reachability, routing, entity linking, movement simulation) into a single mutable singleton that persists for the game session. Every AAS operation—from point-to-area queries to full path planning—begins by reading or updating fields within this world struct. This design enables fast single-threaded access via direct struct field reads but trades flexibility for efficiency, making it difficult to parallelize or hot-reload navigation data.

## Key Cross-References

### Incoming (who depends on this file)

- **All `be_aas_*.c` runtime implementations** (`be_aas_main.c`, `be_aas_sample.c`, `be_aas_route.c`, `be_aas_reach.c`, `be_aas_entity.c`, `be_aas_cluster.c`, `be_aas_move.c`, `be_aas_debug.c`, etc.): Include this header to access `aas_t` struct fields, entity link structures, routing cache format, and physics settings. Every subsystem's `.c` file directly manipulates the global `aasworld` variable declared in `be_aas_main.h`.

- **`be_interface.c`** (botlib public API): Exposes functions like `BotLoadMap`, `BotUpdateEntity`, `BotLibStartFrame` which call into the subsystem functions that dereference `aasworld` and its nested structures.

- **`code/bspc/*.c` (offline AAS compiler)** via stub adapter: Reuses cluster, reachability, and optimize routines. The `BSPCINCLUDE` guard suppresses botlib-only headers; BSPC defines its own entity and linking layers.

- **`code/server/sv_bot.c`**: Bridges server-side bot lifecycle to botlib. Calls `BotLibSetup`, `BotLibStartFrame`, and bot-specific functions that expect `aasworld` to be initialized.

### Outgoing (what this file depends on)

- **`aasfile.h`** (external, not in repo): Provides forward declarations of `aas_bbox_t`, `aas_vertex_t`, `aas_plane_t`, `aas_edge_t`, `aas_edgeindex_t`, `aas_face_t`, `aas_faceindex_t`, `aas_area_t`, `aas_areasettings_t`, `aas_reachability_t`, `aas_node_t`, `aas_portal_t`, `aas_portalindex_t`, `aas_cluster_t` — the binary-serialized AAS file format structures.

- **`be_aas_funcs.h`** (implied from `be_aas_main.h` includes): Provides `aas_entityinfo_t` (public entity state snapshot) and possibly `MAX_TRAVELTYPES` constant.

- **`q_shared.h`** (via transitive includes): `vec3_t`, `qboolean`, `byte`, `MAX_QPATH`.

- **All conditional subsystem headers** (only when `AASINTERN` is `#define`'d, i.e., inside botlib, not BSPC): `be_aas_main.h`, `be_aas_entity.h`, `be_aas_sample.h`, `be_aas_cluster.h`, `be_aas_reach.h`, `be_aas_route.h`, `be_aas_routealt.h`, `be_aas_debug.h`, `be_aas_file.h`, `be_aas_optimize.h`, `be_aas_bsp.h`, `be_aas_move.h`. These declare the internal `AASINTERN` function prototypes that operate on `aasworld`.

## Design Patterns & Rationale

### 1. Monolithic Singleton Pattern
The `aas_t` struct centralizes all mutable AAS state into a single global (`aasworld`). This is idiomatic for id Tech 3 game engines where single-threaded simulation dominates and direct struct field access is faster than indirection. Trade-off: tightly couples all subsystems; makes parallelization and hot-reloading difficult.

### 2. Intrusive Doubly-Linked Lists
`aas_link_t` and `bsp_link_t` use intrusive pointers (`next_ent`, `prev_ent`, `next_area`, `prev_area`), embedding list nodes directly in the structures rather than wrapping them. This saves an allocation per link but requires manual list maintenance. Both entity→area and area→entity mappings share the same nodes, forming a bidirectional graph.

### 3. LRU Routing Cache
`aas_routingcache_t` chains via `time_prev`/`time_next` (oldest-to-newest). When the cache evicts (call to `AAS_FreeOldestCache`), it removes the LRU entry. This minimizes pathfinding recalculation but can thrash under certain query patterns (e.g., bots with uncorrelated goals).

### 4. Variable-Length Data Packing
`aas_routingcache_t` defines `traveltimes[1]` as a 1-element array. At allocation, `sizeof(...) + (actual_count - 1) * sizeof(short)` bytes are reserved, allowing the struct to act as a variable-length record. This is a C idiom predating C99 flexible array members.

### 5. Conditional Compilation for Tool Reuse
`#ifndef BSPCINCLUDE` guards the botlib-specific function declarations. BSPC (the offline compiler) reuses type definitions but has its own function set, avoiding circular dependencies and reducing runtime binary size.

### 6. Physics Settings as Global Lookup Tables
`aas_settings_t` bundles all tunable game physics (gravity, friction, velocities, jump parameters) and per-reachability-type scalars (`rs_*`) into a single struct. This design avoids global scalar variables cluttering the namespace and allows easy save/restore of physics state during development.

## Data Flow Through This File

### Load-Time Flow
1. **Server calls** `BotLoadMap(mapname)` → `AAS_LoadMap()` → `AAS_LoadFiles()` → `AAS_LoadAASFile()` (from `be_aas_file.c`)
2. **File I/O:** Read `.aas` binary file; populate all `aas_t` arrays: bboxes, vertexes, planes, edges, faces, areas, clusters, portals, reachability links.
3. **Initialization:** `AAS_InitClustering()`, `AAS_InitReachability()`, `AAS_InitRouting()` fill in derived data (reversed reachability links, cluster cache structures).
4. **Entity Setup:** `AAS_InitAASLinkHeap()` allocates the `linkheap` pool for later entity-to-area linking.

### Runtime Flow
1. **Entity Update:** `AAS_UpdateEntity(entnum)` → `AAS_EntityBSPData()` snapshots entity AABB → `AAS_LinkEntity()` → inserts `aas_link_t` into both `arealinkedentities` and per-entity `areas` list.
2. **Pathfinding Query:** Bot queries `AAS_RouteToGoalArea(from, to)` → checks `aas_routingcache_t` LRU cache for cluster/area pair → if miss, runs Dijkstra via `AAS_UpdateAreaRoutingCache()`, storing travel times in cache's `traveltimes[]` array.
3. **Movement Simulation:** `AAS_PredictClientMovement()` uses `aas_settings_t` physics constants to simulate jump arcs and check reachability via sweep traces (which call into `aasworld.nodes` BSP tree).

### Shutdown Flow
1. `AAS_Shutdown()` → `AAS_FreeRoutingCaches()` frees all cache entries, unlinks all entities, deallocates `linkheap`.

## Learning Notes

**For Q3 Engine Study:**
- The **AAS system is a precomputed roadmap architecture**: all navigation geometry is baked offline into areas, reachability links, and clusters. This trades offline computation cost for fast runtime queries. Modern engines use navmeshes or RVO obstacles; Q3 uses a hybrid (fixed roadmap + runtime reachability).
- The **entity linking pattern** (`aas_link_t`) is a classic intrusive doubly-linked list used in spatial partitioning. It allows O(1) unlink and relink operations but requires careful pointer management to avoid corruption.
- **Physics settings as a struct** (`aas_settings_t`) contrasts with modern engines' data-driven approaches (YAML, JSON configs). This approach is tight and fast but requires recompilation to tune gameplay.
- The **conditional-compilation pattern** (BSPC vs. botlib) shows how offline tools and runtime engines can share utility code without creating a monolithic binary.

**Idiomatic to This Era:**
- No dynamic memory fragmentation tracking; fixed pools (`linkheap`, reachability heap) allocated once at load time.
- Pointer-heavy code; no reference counting or smart pointers (this predates modern C++ practices by years).
- Global singleton (`aasworld`) is the only mutable state; no thread-local storage or concurrent access (single-threaded assumption).

## Potential Issues

1. **Monolithic Struct Limits Scalability:** The 300+ line `aas_t` struct combines geometry, routing, entity linking, and caching. If one subsystem (e.g., routing cache) consumes excessive memory, it bloats the entire world struct. Fragmentation between subsystems is impossible.

2. **Entity Linking Fragility:** Intrusive doubly-linked lists are error-prone if `aas_link_t` nodes are not carefully unlinked before deallocation or if corruption occurs in the list pointers. No assertions guard against dangling pointers in `next_ent`/`prev_ent`.

3. **Routing Cache Thrashing:** Under pathological query patterns (many bots with uncorrelated goals), the LRU cache may evict frequently-accessed entries, forcing expensive Dijkstra recalculation each frame. The cache size is not tunable per map.

4. **No Invalidation Strategy for Dynamic Geometry:** The `aasworld` assumes static geometry post-load. Moving doors/platforms are handled at runtime via entity collision, but if the map topology changes dynamically (e.g., destructible geometry), reachability links become stale. There is no cache invalidation mechanism.

5. **Hardcoded Physics Constants:** `aas_settings_t` is loaded from the `.aas` file or hardcoded. If a server wants to tweak jump velocity mid-match, all routing caches remain based on the old physics; bots may reach areas that are no longer reachable with new physics.
