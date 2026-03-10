# code/botlib/be_aas_route.c — Enhanced Analysis

## Architectural Role

This file is the **hierarchical pathfinding core** of the entire botlib subsystem. It sits at the intersection of the AAS world representation (loaded by `be_aas_file.c`, clustered by `be_aas_cluster.c`) and the AI decision layer (`be_ai_goal.c`, `be_ai_move.c`), acting as the routing oracle that converts area-pair queries into travel costs and reachability indices. The two-level cluster/portal cache hierarchy means that pathfinding complexity scales with portal count rather than total area count — a key architectural decision enabling the large, complex maps of Q3A. It also bridges offline precomputation (`.rcd` files) with runtime demand-paged caching, making it the file most responsible for bot responsiveness on both first load and during play.

## Key Cross-References

### Incoming (who depends on this file)

- **`be_aas_routealt.c`** (`AAS_AlternativeRouteGoals`) calls `AAS_AreaTravelTimeToGoalArea` to score alternative goal candidates; this file is the only other routing consumer inside botlib.
- **`be_ai_goal.c`** (goal selection) and **`be_ai_move.c`** (movement execution) are the primary per-frame callers of `AAS_AreaTravelTimeToGoalArea` and `AAS_AreaReachabilityToGoalArea` — the two functions that constitute the hot path of bot navigation.
- **`be_aas_main.c`** calls `AAS_InitRouting` during `AAS_ContinueInit` and `AAS_FreeRoutingCaches` during `AAS_Shutdown`; lifecycle is fully mediated through the AAS main module.
- **`be_interface.c`** exposes `AAS_EnableRoutingArea`, `AAS_PredictRoute`, and the travel-time/reachability queries as entries in `botlib_export_t` — the only public face to the game VM.
- **`code/game/g_bot.c`** and the AI FSM in **`ai_dmnet.c`** reach these functions entirely through `trap_BotLib*` syscalls (opcodes 200–599), never linking directly.
- `routingcachesize` and `max_routingcachesize` globals are read by nothing outside this file; they are purely internal diagnostics and pressure valves.

### Outgoing (what this file depends on)

- **`be_aas_sample.c`**: `AAS_PointAreaNum`, `AAS_TraceAreas`, `AAS_TraceClientBBox` — used in `AAS_NearestHideArea` for visibility and area lookup.
- **`be_aas_main.c`**: `AAS_Time`, `AAS_ProjectPointOntoVector` — time-stamping caches and projecting positions in `AAS_NearestHideArea`.
- **`be_aas_reach.c`**: `AAS_AreaCrouch`, `AAS_AreaSwim`, `AAS_AreaGroundFaceArea`, `AAS_AreaDoNotEnter` — used to compute per-area travel time scalars and filter reachability links.
- **`be_aas_bspq3.c`** (via `be_aas_bsp.h`): `AAS_AreaVisible` — visibility check in hide-area search.
- **`l_memory.c`**: `GetMemory`, `GetClearedMemory`, `FreeMemory`, `AvailableMemory` — all heap management; `AvailableMemory` drives the eviction threshold.
- **`l_libvar.c`**: `LibVarValue("max_routingcache", ...)` — reads the only externally tunable parameter for this subsystem.
- **`l_crc.c`**: `CRC_ProcessString` — validates that a `.rcd` file matches the currently loaded AAS world before restoring cached data.
- **`botimport`** vtable: `Print`, `FS_FOpenFile`, `FS_Read`, `FS_Write`, `FS_FCloseFile` for `.rcd` I/O and diagnostics.

## Design Patterns & Rationale

- **Hierarchical pathfinding (pre-HPA\*)**: The two-level cluster/portal split is the same idea later formalized as Hierarchical Path-Finding A\* (Botea et al., 2004). The tradeoff: portal routing adds a conservative over-estimate (`portalmaxtraveltimes`) for cross-cluster paths, accepting suboptimality to avoid a full global search. This was the right call for a 1999 game — exact optimal paths across hundreds of areas per frame per bot were infeasible.
- **Goal-centric (backwards) Dijkstra**: Computing outward from the goal rather than the source means a single `aas_routingcache_t` entry serves every possible source within the cluster. This amortizes the expensive relaxation across all bots querying the same goal.
- **Demand-paged LRU cache**: Cache entries are created on first query and linked into a doubly-linked LRU list anchored at `aasworld.oldestcache`/`newestcache`. Eviction (`AAS_FreeOldestCache`) is triggered by `AvailableMemory() < 1 MB`, not by a fixed count — a fragility: heap fragmentation could trigger premature eviction.
- **Event-driven cache invalidation**: `AAS_EnableRoutingArea` invalidates all caches touching an area when its `AREA_DISABLED` flag changes. This cleanly supports dynamic level geometry (doors, elevators) without polling.
- **Serialization for startup amortization**: `.rcd` files let a server with fixed bot configurations skip all initial Dijkstra passes. The CRC check against area and cluster arrays prevents stale cache use after a map change.

## Data Flow Through This File

```
Bot AI query
  │  (areanum, goalareanum, travelflags)
  ▼
AAS_AreaRouteToGoalArea
  │
  ├─ same cluster? ──► AAS_GetAreaRoutingCache(cluster, goalarea, tfl)
  │                        │ cache miss? → AAS_UpdateAreaRoutingCache
  │                        │   reads: aasworld.reversedreachability (backwards graph)
  │                        │   writes: cache->traveltimes[], cache->reachabilities[]
  │                        └─ returns cache; lookup by clusterareanum
  │
  └─ cross-cluster? ──► AAS_GetPortalRoutingCache(goalarea, tfl)
                            │ cache miss? → AAS_UpdatePortalRoutingCache
                            │   for each portal cluster: AAS_GetAreaRoutingCache
                            │   relaxes portal-level graph
                            └─ for each candidate portal:
                                 portaltime + areacache[portalarea] + conservative overhead
                                 → best (traveltime, reachnum) returned
```

State transitions: areas move from `AREA_DISABLED` ↔ enabled via `AAS_EnableRoutingArea`, which flushes all caches that include that area. Cache entries move from `NULL` → allocated → LRU-linked → possibly evicted (freed + unlinked) as heap pressure grows.

## Learning Notes

- **`unsigned short int` travel times**: Travel times are stored in 16-bit values (max 65535), representing hundredths of a second. At walk speed this caps out at ~655 seconds — well beyond any reachable path on Q3A maps, but the choice encodes an era assumption: memory bandwidth was the bottleneck, not arithmetic range.
- **`__inline` keyword**: Non-standard MSVC extension, not `static inline`. The hot-path helpers (`AAS_ClusterAreaNum`, `AAS_TravelFlagForType_inline`, `AAS_AreaContentsTravelFlags_inline`) use this — modern compilers would use `static inline` or rely on LTO.
- **Reversed reachability graph**: Building `reversedreachability` during init is a classic optimization for goal-directed search. Modern navmesh libraries (Recast/Detour) do the same but in a more general form.
- **`aasworld` global singleton**: All routing state hangs off one global. No per-bot or per-thread state exists — bots share all caches. This makes multi-bot scenarios efficient (shared Dijkstra results) but rules out thread-safe multi-core bot updates.
- **Pre-HPA\* hierarchical nav**: The cluster/portal abstraction is architecturally equivalent to what the academic game-AI literature later called abstract graphs or navigation meshes with portal nodes. Q3A pioneered this in a shipping game engine.

## Potential Issues

- **Bug in `AAS_TravelFlagForType_inline`** (line ~175): `tfl` is initialized to `0`, then immediately tested `if (tfl & TRAVELFLAG_NOTTEAM1)` — this condition is always false. The intent appears to be testing `traveltype & TRAVELFLAG_NOTTEAM1`, but `traveltype` is masked with `TRAVELTYPE_MASK` after this check, stripping those flag bits. Team-exclusion flags on travel types are silently dropped; this is a latent correctness bug for team-mode routing.
- **Conservative cross-cluster travel estimate**: `portalmaxtraveltimes` used in `AAS_AreaRouteToGoalArea` is a worst-case over-estimate, meaning bots may take suboptimal cross-cluster paths when multiple portal options exist. This is documented by design but can produce noticeably poor routing on large interconnected maps.
- **`AvailableMemory() < 1 MB` eviction threshold**: Hardcoded, not tunable. On systems with large heap allocations from other subsystems this threshold may trigger excessive cache eviction, causing repeated Dijkstra recomputations within a single frame.
