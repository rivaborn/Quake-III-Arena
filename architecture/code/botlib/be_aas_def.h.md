# code/botlib/be_aas_def.h

## File Purpose
Central internal definition header for the AAS (Area Awareness System) botlib subsystem. Defines all major data structures, constants, and macros used by the AAS pathfinding and navigation system, and acts as the aggregating include for all AAS subsystem headers.

## Core Responsibilities
- Define the monolithic `aas_t` world state structure holding all AAS geometry, routing, and entity data
- Define entity link structures for spatial partitioning across AAS areas and BSP leaves
- Define routing cache and routing update structures for the pathfinding algorithm
- Define physics and reachability settings via `aas_settings_t`
- Provide entity-index conversion macros (`DF_AASENTNUMBER`, etc.)
- Pull in all AAS subsystem headers when compiled inside the botlib (guarded by `BSPCINCLUDE`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `aas_stringindex_t` | struct | Maps integer indices to string arrays (model/sound/image names) |
| `aas_link_t` | struct | Doubly-linked node linking an entity to an AAS area; forms two interleaved lists (per-entity and per-area) |
| `bsp_link_t` | struct | Same dual-linked structure, but for BSP leaves instead of AAS areas |
| `bsp_entdata_t` | struct | Collision-relevant snapshot of an entity (origin, angles, AABB, solid, modelnum) |
| `aas_entity_t` | struct | Full AAS entity: wraps `aas_entityinfo_t` plus area-link and leaf-link list heads |
| `aas_settings_t` | struct | Physics constants (gravity, friction, velocities, step heights) and per-reachability-type risk/cost scalars |
| `aas_routingcache_t` | struct | Variable-length cache entry storing precomputed travel times per cluster/area; LRU-linked |
| `aas_routingupdate_t` | struct | Per-area working node for Dijkstra-style routing propagation |
| `aas_reversedlink_t` | struct | Single reversed reachability edge (used for backwards graph traversal) |
| `aas_reversedreachability_t` | struct | List head of reversed reachability edges for one area |
| `aas_reachabilityareas_t` | struct | Range of areas a reachability transition passes through |
| `aas_t` | struct | Master AAS world: all geometry arrays, routing caches, entity tables, cluster/portal data, and runtime state |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `aasworld` | `aas_t` (extern, declared in `be_aas_main.h`) | global | Singleton AAS world state; all AAS subsystems read/write through this |

## Key Functions / Methods

No functions are defined in this header. It is a pure type/macro/constant definition file plus a bundled include aggregator.

- **Notes:** All function declarations are deferred to the subsystem headers conditionally included at the bottom (`be_aas_main.h`, `be_aas_entity.h`, `be_aas_sample.h`, `be_aas_cluster.h`, `be_aas_reach.h`, `be_aas_route.h`, `be_aas_routealt.h`, `be_aas_debug.h`, `be_aas_file.h`, `be_aas_optimize.h`, `be_aas_bsp.h`, `be_aas_move.h`). Internal-only symbols are further gated behind `#ifdef AASINTERN` in each child header.

## Control Flow Notes

This header is included at the top of every AAS `.c` implementation file. It establishes the complete type environment before any function definitions. The `#define AASINTERN` immediately before the conditional includes enables the `AASINTERN`-gated internal function declarations (e.g., `AAS_Setup`, `AAS_Shutdown`, `AAS_StartFrame`) in the bundled headers. `BSPCINCLUDE` suppresses the botlib-specific headers when the file is compiled inside the BSPC map compiler, which reuses AAS types but has its own function set.

## External Dependencies

- `aasfile.h` (implied): provides `aas_bbox_t`, `aas_vertex_t`, `aas_plane_t`, `aas_edge_t`, `aas_face_t`, `aas_area_t`, `aas_areasettings_t`, `aas_reachability_t`, `aas_node_t`, `aas_portal_t`, `aas_cluster_t`, and related index/count types — defined elsewhere
- `be_aas_funcs.h` / `be_aas.h` (implied): provides `aas_entityinfo_t`, `aas_trace_t`, `aas_areainfo_t`, `bot_entitystate_t`, `MAX_TRAVELTYPES` — defined elsewhere
- `q_shared.h` (implied): `vec3_t`, `qboolean`, `MAX_QPATH`, `byte` — defined elsewhere
- All `be_aas_*.h` subsystem headers: forward-declare the internal and public AAS API; implementations are defined in their corresponding `.c` files
