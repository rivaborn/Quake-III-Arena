# code/botlib/be_aas_cluster.c

## File Purpose
Implements AAS (Area Awareness System) area clustering for the Quake III bot navigation library. It partitions the navigable world into clusters separated by portal areas, enabling efficient hierarchical pathfinding by reducing the search space within each cluster.

## Core Responsibilities
- Identify and create portal areas that act as boundaries between clusters
- Flood-fill cluster membership from seed areas using reachability links (and optionally face adjacency)
- Assign cluster-local area indices to all areas and portals within each cluster
- Detect and heuristically generate candidate portal areas (`AAS_FindPossiblePortals`)
- Validate that every portal has exactly one front and one back cluster
- Initialize and reinitialize the full clustering data structures (`AAS_InitClustering`)
- Manage view portals (a superset of cluster portals used for PVS)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `aas_portal_t` | struct (defined in `be_aas_def.h`) | Represents a portal area with front/back cluster refs and per-cluster area numbers |
| `aas_cluster_t` | struct (defined in `be_aas_def.h`) | Stores area count, reachability area count, and portal index range for a cluster |
| `aas_area_t` | struct (defined in `aasfile.h`) | BSP area with face list |
| `aas_face_t` | struct (defined in `aasfile.h`) | Face with front/back area references and flags |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `nofaceflood` | `int` | file-global | When `qtrue`, cluster flooding ignores face adjacency and only uses reachability links |
| `botimport` | `botlib_import_t` (extern) | global | Engine import table for printing, error reporting |
| `aasworld` | (extern, defined in `be_aas_main.c`) | global singleton | Entire loaded AAS world state: areas, faces, portals, clusters, reachabilities |

## Key Functions / Methods

### AAS_InitClustering
- **Signature:** `void AAS_InitClustering(void)`
- **Purpose:** Top-level entry point. Reinitializes all portal and cluster data, then iterates until a valid clustering is found.
- **Inputs:** None (reads/writes `aasworld`, reads `LibVar` cvars `forceclustering`/`forcereachability`)
- **Outputs/Return:** None
- **Side effects:** Frees and reallocates `aasworld.portals`, `aasworld.portalindex`, `aasworld.clusters`; sets `aasworld.savefile = qtrue`; prints stats via `botimport.Print`
- **Calls:** `AAS_SetViewPortalsAsClusterPortals`, `AAS_CountForcedClusterPortals`, `AAS_RemoveClusterAreas`, `AAS_FindPossiblePortals`, `AAS_CreateViewPortals`, `FreeMemory`, `GetClearedMemory`, `AAS_CreatePortals`, `AAS_FindClusters`, `AAS_TestPortals`, `Log_Write`, `botimport.Print`
- **Notes:** Runs a `while(1)` retry loop — if `AAS_FindClusters` or `AAS_TestPortals` fails, it retries (currently the portal removal logic inside the loop is incomplete/stubbed — `removedPortalAreas` increments but never actually removes a portal before retry).

### AAS_FindClusters
- **Signature:** `int AAS_FindClusters(void)`
- **Purpose:** Seeds and floods all clusters by iterating over unassigned areas.
- **Inputs:** None
- **Outputs/Return:** `qtrue` on success, `qfalse` on error
- **Side effects:** Modifies `aasworld.areasettings[*].cluster`, `aasworld.clusters[*]`, `aasworld.numclusters`
- **Calls:** `AAS_RemoveClusterAreas`, `AAS_FloodClusterAreas_r`, `AAS_FloodClusterAreasUsingReachabilities`, `AAS_NumberClusterAreas`, `AAS_Error`

### AAS_FloodClusterAreas_r
- **Signature:** `int AAS_FloodClusterAreas_r(int areanum, int clusternum)`
- **Purpose:** Recursive DFS flood-fill assigning `clusternum` to `areanum` and all areas reachable from it.
- **Inputs:** `areanum` — area to process; `clusternum` — cluster being built
- **Outputs/Return:** `qtrue` on success, `qfalse` on conflict/error
- **Side effects:** Sets `aasworld.areasettings[areanum].cluster` and `.clusterareanum`; increments `aasworld.clusters[clusternum].numareas`; calls `AAS_UpdatePortal` for portal areas
- **Calls:** `AAS_UpdatePortal`, `AAS_FloodClusterAreas_r` (recursive), `AAS_Error`
- **Notes:** Face-based flooding is guarded by `!nofaceflood`; reachability-based flooding always runs.

### AAS_UpdatePortal
- **Signature:** `int AAS_UpdatePortal(int areanum, int clusternum)`
- **Purpose:** Assigns `clusternum` as front or back cluster of the portal at `areanum`; adds portal to cluster's portal index.
- **Inputs:** `areanum` — portal area; `clusternum` — cluster touching this portal
- **Outputs/Return:** `qtrue` if portal is already fully updated or successfully updated; `qfalse` if a portal would separate more than 2 clusters
- **Side effects:** Writes `portal->frontcluster`/`backcluster`; modifies `aasworld.portalindex`; increments `portalindexsize` and `cluster->numportals`; sets `areasettings[areanum].cluster` to negative portal number
- **Calls:** `AAS_Error`, `Log_Write`

### AAS_NumberClusterAreas
- **Signature:** `void AAS_NumberClusterAreas(int clusternum)`
- **Purpose:** Assigns cluster-local sequential indices to areas and portals within a cluster, with reachability areas numbered first.
- **Inputs:** `clusternum`
- **Outputs/Return:** None
- **Side effects:** Resets and repopulates `aasworld.clusters[clusternum].numareas`/`numreachabilityareas`; updates `areasettings[*].clusterareanum` and `portal->clusterareanum[0/1]`
- **Calls:** `AAS_AreaReachability`

### AAS_CheckAreaForPossiblePortals
- **Signature:** `int AAS_CheckAreaForPossiblePortals(int areanum)`
- **Purpose:** Heuristically tests whether an area (and adjacent areas with fewer presence types) qualifies as a cluster portal based on geometric criteria (exactly two planar sides, connected front/back neighborhoods, no shared edges between sides).
- **Inputs:** `areanum`
- **Outputs/Return:** Number of areas marked as portals (0 if not a portal)
- **Side effects:** Sets `AREACONTENTS_CLUSTERPORTAL | AREACONTENTS_ROUTEPORTAL` on qualifying areas; calls `Log_Write`
- **Calls:** `AAS_GetAdjacentAreasWithLessPresenceTypes_r`, `AAS_ConnectedAreas`, `AAS_Error`, `Log_Write`

### AAS_TestPortals
- **Signature:** `int AAS_TestPortals(void)`
- **Purpose:** Validates that every portal has both a front and back cluster assigned.
- **Inputs:** None
- **Outputs/Return:** `qtrue` if all portals valid, `qfalse` otherwise
- **Side effects:** Clears `AREACONTENTS_CLUSTERPORTAL` on invalid portals; writes to log

### Notes on minor helpers
- `AAS_RemoveClusterAreas` / `AAS_ClearCluster` — zero out cluster assignments
- `AAS_RemovePortalsClusterReference` — clears a cluster's reference from portals
- `AAS_CreatePortals` — allocates portal structs from portal-content areas
- `AAS_FindPossiblePortals` — iterates all areas calling `AAS_CheckAreaForPossiblePortals`
- `AAS_ConnectedAreas` / `AAS_ConnectedAreas_r` — checks face-based connectivity among a set of areas
- `AAS_CreateViewPortals` / `AAS_SetViewPortalsAsClusterPortals` — sync view portals ↔ cluster portals
- `AAS_FloodClusterAreasUsingReachabilities` — secondary pass to pull isolated areas into a cluster via reachability links
- Large `#if 0` block contains dead/experimental code for teleporter portals and alternative non-convex merging checks.

## Control Flow Notes
`AAS_InitClustering` is called once during AAS world initialization (triggered by `be_aas_main.c`). It is not called per-frame. The clustering is a precomputation step: results are stored in `aasworld` and the AAS file is flagged for saving. The cluster data is subsequently consumed by the routing system (`be_aas_route.c`) for hierarchical A\* pathfinding.

## External Dependencies
- **Includes:** `q_shared.h`, `l_memory.h`, `l_script.h`, `l_precomp.h`, `l_struct.h`, `l_log.h`, `l_libvar.h`, `aasfile.h`, `botlib.h`, `be_aas.h`, `be_aas_funcs.h`, `be_aas_def.h`
- **Defined elsewhere:**
  - `aasworld` — global AAS world state (`be_aas_main.c`)
  - `botimport` — engine import struct (`be_interface.c`)
  - `AAS_AreaReachability` — reachability query (`be_aas_reach.c` / `be_aas_funcs.h`)
  - `AAS_Error`, `Log_Write` — error/log utilities (`be_aas_main.c`, `l_log.c`)
  - `GetClearedMemory`, `FreeMemory` — bot memory allocator (`l_memory.c`)
  - `LibVarGetValue` — library variable query (`l_libvar.c`)
