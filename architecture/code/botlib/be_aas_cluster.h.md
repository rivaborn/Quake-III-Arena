# code/botlib/be_aas_cluster.h

## File Purpose
Header file for the AAS (Area Awareness System) clustering subsystem within the botlib. It declares internal functions for initializing area clusters and designating view portals as cluster portals, guarded behind the `AASINTERN` preprocessor gate.

## Core Responsibilities
- Declares the clustering initialization entry point for internal AAS use
- Declares the function to promote view portals to cluster portal status
- Guards all declarations behind `AASINTERN` so they are invisible to external callers

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_InitClustering
- **Signature:** `void AAS_InitClustering(void)`
- **Purpose:** Initializes the AAS area clustering system, partitioning navigable areas into clusters for hierarchical pathfinding.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Mutates global AAS world state (cluster data) — defined elsewhere in `be_aas_cluster.c`.
- **Calls:** Not inferable from this file.
- **Notes:** Only callable from within the botlib (`AASINTERN` scope). Must be called during AAS world load/init before routing queries are issued.

### AAS_SetViewPortalsAsClusterPortals
- **Signature:** `void AAS_SetViewPortalsAsClusterPortals(void)`
- **Purpose:** Marks existing view portals in the AAS data as cluster portals, making them traversal boundaries between clusters.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Mutates portal/cluster flags in global AAS world state.
- **Calls:** Not inferable from this file.
- **Notes:** Only callable from within the botlib (`AASINTERN` scope). Likely called as part of or immediately after `AAS_InitClustering`.

## Control Flow Notes
Both functions are part of the AAS world **initialization** path. They are invoked at map load time (not per-frame) to build the cluster topology that the routing system (`be_aas_route.c`) depends on for hierarchical bot navigation.

## External Dependencies
- No includes in this file.
- `AASINTERN` macro — defined externally (likely in `be_aas_def.h` or a compilation unit that includes internal AAS headers) to gate visibility of these declarations.
- Implementation defined in `code/botlib/be_aas_cluster.c`.
