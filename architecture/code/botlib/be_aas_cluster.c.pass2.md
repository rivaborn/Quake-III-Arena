# code/botlib/be_aas_cluster.c — Enhanced Analysis

## Architectural Role

This file implements the **second level of hierarchical spatial decomposition** in the AAS navigation system. Whereas individual areas represent micro-scale navigable regions, clusters group connected areas into macro-scale regions separated by portal boundaries. This two-level hierarchy is consumed by `be_aas_route.c` to perform efficient pathfinding: inter-cluster routing via portals (cross-cluster edges) followed by intra-cluster routing within each destination cluster. The clustering precomputation is not per-frame; it runs once during `AAS_Setup()` / `AAS_ContinueInit()` and the results are persisted to the `.aas` file.

## Key Cross-References

### Incoming (who depends on this file)

- **`be_aas_main.c`**: Calls `AAS_InitClustering()` during initial/incremental AAS load to populate cluster and portal data structures
- **`be_aas_route.c`**: Reads `aasworld.clusters[]` and `aasworld.portals[]` to execute hierarchical A\* queries (`AAS_AreaRouteToGoalArea`, cache lookups, portal traversal)
- **`be_aas.h`** / **public botlib API**: Cluster data is indirectly exposed to the game/server via routing queries; the server uses hierarchical routing for bot pathfinding
- **`bspc/be_aas_bspc.c`**: Offline AAS compiler reuses this module's clustering logic (stub import wrapper)

### Outgoing (what this file depends on)

- **`be_aas_funcs.h`** / **`be_aas_reach.c`**: `AAS_AreaReachability()` queries which areas have reachable neighbors (used to distinguish high-value vs. dead-end areas during numbering)
- **`be_aas_main.c`**: `AAS_Error()` for fatal errors, `AAS_ProjectPointOntoVector()` for geometry (note: not called in current code but declared in header)
- **`l_log.c`**: `Log_Write()` for structured logging of clustering decisions (portal separation, invalid portals)
- **`l_memory.c`**: `GetClearedMemory()`, `FreeMemory()` for dynamic allocation of portal index buffers
- **`l_libvar.c`**: `LibVarGetValue()` to read cvars `forceclustering` and `forcereachability` (retry policy)
- **`q_shared.h`**: Fundamental types (`vec_t`, `vec3_t`, `qtrue`/`qfalse`)
- **Global `aasworld`** (defined in `be_aas_main.c`): Reads/writes all cluster, portal, area, reachability, and face data

## Design Patterns & Rationale

**Hierarchical graph partitioning** — The cluster/portal model is a lightweight implementation of a **cut-based hierarchy**: portals are the "edges" connecting clusters (the "super-nodes"). This reduces pathfinding complexity from O(areas) to O(clusters × cluster_size).

**Greedy flood-fill with heuristic seeding** — `AAS_FindClusters()` iterates over unassigned areas, growing each new cluster via recursive DFS. The choice to seed clusters from areas with reachabilities (when `nofaceflood`) biases toward navigation-friendly starting points, avoiding dead-end areas.

**Dual-numbering scheme** — `AAS_NumberClusterAreas()` assigns **global area indices** (for external references) *and* **per-cluster local indices** (for cache locality). Reachable areas are numbered before unreachable ones, prioritizing the "navigable core" of each cluster.

**Incomplete retry-on-failure pattern** — `AAS_InitClustering()` contains a `while(1)` loop that retries clustering after removing a problematic portal. However, the actual portal removal (`// TODO: remove portal`) is stubbed out, suggesting this was a deferred improvement or the logic was never fully implemented.

**Configurable flood semantics** — The `nofaceflood` flag allows clustering to ignore BSP face adjacency and rely solely on reachability links. This trade-off prioritizes pathfinding correctness over strict geometric connectivity.

## Data Flow Through This File

**Input flow:**
1. AAS world loaded: areas, faces, reachability links, BSP data populated by `AAS_LoadBSPFile` / `AAS_ContinueInitReachability`
2. Portal candidates identified: `AAS_FindPossiblePortals()` marks AREACONTENTS_CLUSTERPORTAL on qualifying areas
3. Portal objects created: `AAS_CreatePortals()` allocates `aas_portal_t` structs and initializes a cluster-index array

**Processing:**
1. **Seed clusters**: `AAS_FindClusters()` iterates areas, creating cluster 0, 1, 2, … as it discovers unassigned areas
2. **Flood membership**: `AAS_FloodClusterAreas_r()` recursively assigns areas to the current cluster; when it hits a portal, calls `AAS_UpdatePortal()` to log which cluster it borders
3. **Assign indices**: `AAS_NumberClusterAreas()` renumbers all areas within a cluster with sequential local indices (reachable first)
4. **Validate**: `AAS_TestPortals()` ensures no portal was touched by more than 2 clusters (catches geometric contradictions)

**Output flow:**
1. `aasworld.areasettings[areanum].cluster` — set to positive cluster ID (normal area) or negative portal number (portal area)
2. `aasworld.clusters[clusternum]` — populated with area count, reachability area count, and portal range in `portalindex`
3. `aasworld.portals[portalnum]` — front/back cluster assignments set; per-cluster area numbers recorded
4. `aasworld.portalindex[]` — sparse array of portal numbers indexed by cluster
5. **Persistence**: `AAS_WriteAASFile()` serializes the clustering; on next load, `AAS_LoadAASFile()` restores it

## Learning Notes

**Spatial hierarchy as a routing optimization**: Modern game engines use similar concepts (waypoint networks, nav meshes, hierarchical pathfinding). Quake III's cluster/portal model is a simple but effective precursor.

**Heuristic geometry analysis**: `AAS_CheckAreaForPossiblePortals()` uses adjacency counts and connectivity checks to auto-detect bottle-neck areas—this is ad-hoc ML-like feature engineering before machine learning was common in game dev.

**Floating-point geometry in navigation**: The file demonstrates care with planar geometry (face normals, point-on-plane tests) to distinguish "sides" of a portal, anticipating later non-convex portal merging (commented out).

**Early 2000s debugging approach**: Extensive `Log_Write()` calls and the `#if 0` dead code blocks suggest this was refined through iterative testing and bugfixes, with features (teleporter portals, non-convex merging) deferred when they proved complex.

## Potential Issues

- **Incomplete retry logic**: The while-loop in `AAS_InitClustering` is unreachable in normal operation; `removedPortalAreas` is incremented but the portal is never actually removed before retrying, making the retry loop a no-op.
- **Silent cluster boundary violations**: If `AAS_FloodClusterAreas_r` encounters a cluster boundary during reachability traversal (i.e., a reachability arrow crosses two clusters), it logs an error but the flood can still succeed, potentially leaving the world in an inconsistent state.
- **Tight coupling to reachability timing**: Clustering depends on reachability computation completing first. If reachabilities are incomplete or invalid, clustering will produce suboptimal results.
