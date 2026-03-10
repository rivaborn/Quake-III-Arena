# code/botlib/be_aas_route.c

## File Purpose
Implements the AAS (Area Awareness System) pathfinding and routing subsystem for the Quake III bot library. It computes and caches travel times between areas and portals using a Dijkstra-like relaxation algorithm over the cluster/portal graph. It also provides route prediction and nearest-hide-area queries for AI decision-making.

## Core Responsibilities
- Initialize and tear down all routing data structures (area/portal caches, travel time tables, reversed reachability links)
- Compute intra-cluster travel times via `AAS_UpdateAreaRoutingCache` (BFS/relaxation over reversed reachability)
- Compute inter-cluster travel times via `AAS_UpdatePortalRoutingCache` (portal-level routing)
- Manage an LRU-ordered routing cache with size limits; evict oldest entries when memory is low
- Serialize/deserialize pre-computed routing caches to/from `.rcd` files
- Expose `AAS_AreaTravelTimeToGoalArea` and `AAS_AreaReachabilityToGoalArea` as the primary bot query API
- Predict a route step-by-step with configurable stop events (`AAS_PredictRoute`)
- Find the nearest area hidden from an enemy (`AAS_NearestHideArea`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `routecacheheader_t` | struct | File header for serialized `.rcd` route cache dumps; includes CRCs for validation |
| `aas_routingcache_t` | struct (defined in `be_aas_def.h`) | Per-(cluster,area,travelflags) cache storing travel times and reachability indices; linked into LRU list |
| `aas_routingupdate_t` | struct (defined elsewhere) | Temporary node used during the relaxation update queue |
| `aas_reversedreachability_t` / `aas_reversedlink_t` | structs (defined elsewhere) | Reversed adjacency list used to propagate costs backwards from goal |
| `aas_reachabilityareas_t` | struct (defined elsewhere) | Lists AAS areas a reachability link passes through, used by `AAS_PredictRoute` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `numareacacheupdates` | `int` | global (ROUTING_DEBUG only) | Counter of area cache recomputations for diagnostics |
| `numportalcacheupdates` | `int` | global (ROUTING_DEBUG only) | Counter of portal cache recomputations for diagnostics |
| `routingcachesize` | `int` | global | Running total bytes currently allocated to routing caches |
| `max_routingcachesize` | `int` | global | Maximum allowed routing cache size in bytes (from libvar `max_routingcache`) |

## Key Functions / Methods

### AAS_InitRouting
- **Signature:** `void AAS_InitRouting(void)`
- **Purpose:** Master initialization entry point; sets up all routing tables and reads any cached `.rcd` file.
- **Inputs:** None (reads `aasworld` global)
- **Outputs/Return:** void
- **Side effects:** Allocates all routing arrays into `aasworld`; sets `routingcachesize = 0`; reads `max_routingcache` libvar
- **Calls:** `AAS_InitTravelFlagFromType`, `AAS_InitAreaContentsTravelFlags`, `AAS_InitRoutingUpdate`, `AAS_CreateReversedReachability`, `AAS_InitClusterAreaCache`, `AAS_InitPortalCache`, `AAS_CalculateAreaTravelTimes`, `AAS_InitPortalMaxTravelTimes`, `AAS_InitReachabilityAreas`, `AAS_ReadRouteCache`

### AAS_FreeRoutingCaches
- **Signature:** `void AAS_FreeRoutingCaches(void)`
- **Purpose:** Full shutdown/cleanup of all routing memory.
- **Side effects:** Frees all dynamically allocated routing arrays, NULLs pointers in `aasworld`

### AAS_UpdateAreaRoutingCache
- **Signature:** `void AAS_UpdateAreaRoutingCache(aas_routingcache_t *areacache)`
- **Purpose:** Dijkstra-like backwards relaxation within a single cluster from the goal area outward; fills `areacache->traveltimes[]` and `areacache->reachabilities[]`.
- **Inputs:** `areacache` — partially initialized cache with goal area, cluster, and travel flag constraints
- **Side effects:** Writes into `areacache->traveltimes` and `areacache->reachabilities`; reads/writes `aasworld.areaupdate` scratch array; increments `numareacacheupdates`

### AAS_GetAreaRoutingCache
- **Signature:** `aas_routingcache_t *AAS_GetAreaRoutingCache(int clusternum, int areanum, int travelflags)`
- **Purpose:** Look up or create/compute the routing cache for a goal area within a cluster. Manages LRU list linkage.
- **Inputs:** cluster, area, travel flags
- **Outputs/Return:** pointer to valid `aas_routingcache_t`
- **Side effects:** May allocate new cache, call `AAS_UpdateAreaRoutingCache`, update LRU list via `AAS_UnlinkCache`/`AAS_LinkCache`

### AAS_UpdatePortalRoutingCache
- **Signature:** `void AAS_UpdatePortalRoutingCache(aas_routingcache_t *portalcache)`
- **Purpose:** Relaxation pass over the portal graph to compute travel times from all portals to a goal area, using intra-cluster caches as edge weights.
- **Calls:** `AAS_GetAreaRoutingCache` recursively per cluster

### AAS_AreaRouteToGoalArea
- **Signature:** `int AAS_AreaRouteToGoalArea(int areanum, vec3_t origin, int goalareanum, int travelflags, int *traveltime, int *reachnum)`
- **Purpose:** Core routing query — returns the best travel time and first reachability index from `areanum` to `goalareanum`. Handles same-cluster fast path and cross-cluster portal search.
- **Outputs/Return:** `qtrue` on success; fills `*traveltime`, `*reachnum`
- **Side effects:** May evict old caches (`AAS_FreeOldestCache`) if memory is low; calls `AAS_GetAreaRoutingCache` and `AAS_GetPortalRoutingCache`
- **Notes:** Does NOT guarantee optimal cross-cluster routes (adds `portalmaxtraveltimes` as a conservative over-estimate)

### AAS_AreaTravelTimeToGoalArea / AAS_AreaReachabilityToGoalArea
- **Signature:** `int AAS_AreaTravelTimeToGoalArea(...)` / `int AAS_AreaReachabilityToGoalArea(...)`
- **Purpose:** Thin wrappers over `AAS_AreaRouteToGoalArea` returning only travel time or reachability number respectively.

### AAS_PredictRoute
- **Signature:** `int AAS_PredictRoute(struct aas_predictroute_s *route, int areanum, vec3_t origin, int goalareanum, int travelflags, int maxareas, int maxtime, int stopevent, int stopcontents, int stoptfl, int stopareanum)`
- **Purpose:** Simulates stepping along the route, checking for stop events (entering a content type, entering a specific area, using a forbidden travel type).
- **Outputs/Return:** `qtrue` if goal reached; fills `route->stopevent`, `endarea`, `endpos`, `time`

### AAS_NearestHideArea
- **Signature:** `int AAS_NearestHideArea(int srcnum, vec3_t origin, int areanum, int enemynum, vec3_t enemyorigin, int enemyareanum, int travelflags)`
- **Purpose:** Dijkstra search outward from `areanum` to find the closest area not visible from `enemyareanum`, while penalizing paths that move toward the enemy.
- **Side effects:** Uses a static `hidetraveltimes` array (persistent across calls); uses `aasworld.areaupdate` scratch buffer

### AAS_FreeOldestCache
- **Signature:** `int AAS_FreeOldestCache(void)`
- **Purpose:** Evict the oldest non-portal-goal cache entry from the LRU list to reclaim memory.
- **Outputs/Return:** `qtrue` if a cache was freed

### AAS_WriteRouteCache / AAS_ReadRouteCache
- **Purpose:** Serialize all current routing caches to `maps/<mapname>.rcd` / deserialize them on load. Validated via CRC of area and cluster arrays.

### Notes
- `AAS_ClusterAreaNum`, `AAS_TravelFlagForType_inline`, `AAS_AreaContentsTravelFlags_inline` are `__inline` helpers used heavily in hot paths.
- `AAS_CreateReversedReachability` and `AAS_CalculateAreaTravelTimes` are one-time preprocessing steps called during init.

## Control Flow Notes
- Called during bot library initialization (`AAS_InitRouting`) and shutdown (`AAS_FreeRoutingCaches`).
- Per-frame: `AAS_AreaTravelTimeToGoalArea` and `AAS_AreaReachabilityToGoalArea` are called each frame per bot query; caches absorb the cost of repeated queries.
- Cache eviction (`AAS_FreeOldestCache`) is demand-driven within routing queries when heap headroom < 1 MB.
- `AAS_EnableRoutingArea` invalidates caches when area disabled-state changes (event-driven, not per-frame).

## External Dependencies
- **Includes:** `q_shared.h`, `l_utils.h`, `l_memory.h`, `l_log.h`, `l_crc.h`, `l_libvar.h`, `l_script.h`, `l_precomp.h`, `l_struct.h`, `aasfile.h`, `botlib.h`, `be_aas.h`, `be_aas_funcs.h`, `be_interface.h`, `be_aas_def.h`
- **Defined elsewhere:** `aasworld` (global AAS world state), `botimport` (engine I/O interface), `bot_developer` (cvar), `AAS_Time`, `AAS_TraceAreas`, `AAS_TraceClientBBox`, `AAS_PointAreaNum`, `AAS_AreaReachability`, `AAS_AreaCrouch`, `AAS_AreaSwim`, `AAS_AreaGroundFaceArea`, `AAS_AreaDoNotEnter`, `AAS_ProjectPointOntoVector`, `AAS_AreaVisible`, `LibVarValue`, `Sys_MilliSeconds`, `CRC_ProcessString`, `GetMemory`, `GetClearedMemory`, `FreeMemory`, `AvailableMemory`, `Log_Write`
