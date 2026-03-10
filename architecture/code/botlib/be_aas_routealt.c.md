# code/botlib/be_aas_routealt.c

## File Purpose
Implements alternative routing goal discovery for the AAS (Area Awareness System) bot navigation library. It identifies mid-range waypoint areas between a start and goal position that a bot could route through to take paths different from the shortest route.

## Core Responsibilities
- Identify "mid-range" AAS areas that lie geometrically between start and goal positions using travel-time thresholds
- Flood-fill connected mid-range areas into spatial clusters via `AAS_AltRoutingFloodCluster_r`
- Select one representative area per cluster (closest to cluster centroid) as an alternative route goal
- Populate an output array of `aas_altroutegoal_t` structs with alternative waypoints
- Manage lifecycle (init/shutdown) of working buffers `midrangeareas` and `clusterareas`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `midrangearea_t` | struct | Per-area validity flag plus cached start/goal travel times (as `unsigned short`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `midrangeareas` | `midrangearea_t *` | global | Working buffer; one entry per AAS area; populated each call to `AAS_AlternativeRouteGoals` |
| `clusterareas` | `int *` | global | Scratch buffer holding area indices of the current flood-fill cluster |
| `numclusterareas` | `int` | global | Count of areas accumulated in `clusterareas` during a single flood-fill pass |

## Key Functions / Methods

### AAS_AltRoutingFloodCluster_r
- **Signature:** `void AAS_AltRoutingFloodCluster_r(int areanum)`
- **Purpose:** Recursive flood-fill that collects all spatially connected mid-range areas into `clusterareas[]`, consuming (invalidating) each visited area.
- **Inputs:** `areanum` — AAS area index to process.
- **Outputs/Return:** void; appends to globals `clusterareas` / `numclusterareas`, clears `midrangeareas[areanum].valid`.
- **Side effects:** Modifies `midrangeareas`, `clusterareas`, `numclusterareas`. Reads `aasworld.areas` and `aasworld.faces`/`aasworld.faceindex`.
- **Calls:** Itself (recursive), accesses `aasworld` global.
- **Notes:** No depth limit; deep recursion is possible in large open areas. Terminates when no adjacent valid mid-range area exists.

---

### AAS_AlternativeRouteGoals
- **Signature:** `int AAS_AlternativeRouteGoals(vec3_t start, int startareanum, vec3_t goal, int goalareanum, int travelflags, aas_altroutegoal_t *altroutegoals, int maxaltroutegoals, int type)`
- **Purpose:** Main entry point. Computes up to `maxaltroutegoals` alternative routing waypoints between `start` and `goal` by finding mid-range AAS areas, clustering them, and selecting cluster centroids.
- **Inputs:** Start/goal positions and area numbers; travel flags; output buffer and capacity; `type` bitmask (`ALTROUTEGOAL_ALL`, `ALTROUTEGOAL_CLUSTERPORTALS`, `ALTROUTEGOAL_VIEWPORTALS`) controlling which area types are eligible.
- **Outputs/Return:** Number of alternative goals written into `altroutegoals[]`. Returns 0 if disabled or invalid areas.
- **Side effects:** Writes `midrangeareas[]` (zeroed then populated), calls `AAS_AltRoutingFloodCluster_r` which clobbers `clusterareas`/`numclusterareas`. Calls `Log_Write` for each mid-range area found. Optional `AAS_ShowAreaPolygons` in debug builds.
- **Calls:** `AAS_AreaTravelTimeToGoalArea`, `AAS_AreaReachability`, `AAS_AltRoutingFloodCluster_r`, `Com_Memset`, `Log_Write`, vector macros (`VectorClear`, `VectorAdd`, `VectorScale`, `VectorSubtract`, `VectorLength`, `VectorCopy`).
- **Notes:** Mid-range threshold: `starttime ≤ 1.1 × goaltraveltime` AND `goaltime ≤ 0.8 × goaltraveltime`. The stored `starttime`/`goaltime` fields are `unsigned short`, so travel times exceeding 65535 will silently overflow.

---

### AAS_InitAlternativeRouting
- **Signature:** `void AAS_InitAlternativeRouting(void)`
- **Purpose:** Allocates (or reallocates) `midrangeareas` and `clusterareas` based on `aasworld.numareas`.
- **Side effects:** Heap allocation via `GetMemory`; frees previous buffers if non-NULL.
- **Calls:** `FreeMemory`, `GetMemory`.

---

### AAS_ShutdownAlternativeRouting
- **Signature:** `void AAS_ShutdownAlternativeRouting(void)`
- **Purpose:** Frees `midrangeareas` and `clusterareas` and resets all three globals to NULL/0.
- **Side effects:** Heap free; nulls globals.
- **Calls:** `FreeMemory`.

## Control Flow Notes
- `AAS_InitAlternativeRouting` is called during AAS world load (init phase).
- `AAS_AlternativeRouteGoals` is called on-demand by bot AI logic when a bot seeks a flanking or non-direct path.
- `AAS_ShutdownAlternativeRouting` is called during AAS world unload (shutdown phase).
- The entire implementation is conditionally compiled under `#define ENABLE_ALTROUTING`; removing the define reduces all functions to stubs.

## External Dependencies
- `q_shared.h` — vector math macros, `qboolean`, `Com_Memset`
- `l_memory.h` — `GetMemory`, `FreeMemory`
- `l_log.h` — `Log_Write`
- `be_aas_def.h` — `aasworld` global (type `aas_t`), `aas_area_t`, `aas_face_t`
- `be_aas_funcs.h` — `AAS_AreaTravelTimeToGoalArea`, `AAS_AreaReachability`, `AAS_ShowAreaPolygons` (debug)
- `be_interface.h` — `botimport` (used in debug timing path only)
- `aasfile.h` — AAS file structures (`aas_area_t`, `aas_face_t`, area content flags)
- `botlib.h` / `be_aas.h` — `aas_altroutegoal_t`, `ALTROUTEGOAL_*` constants
- `aasworld` — defined in `be_aas_main.c` (external global)
