# code/botlib/aasfile.h — Enhanced Analysis

## Architectural Role

This header defines the **binary interchange format** for Quake III's Area Awareness System (AAS), bridging offline map compilation (`bspc`) and runtime bot navigation (`botlib`). It encodes a hierarchical spatial subdivision (areas grouped into clusters), directional reachability graph (travel modes between areas), and geometric BSP data. The format is specifically tailored to support 14+ distinct bot movement modes (walk, crouch, jump, ladder, rocketjump, grapple, teleport, etc.) and collision classification via presence types.

## Key Cross-References

### Incoming (who depends on this file)

- **Offline: `bspc/aas_file.c`** — writes `.aas` files; shares `AAS_WriteAASFile`, `AAS_LoadAASFile`, `AAS_SwapAASData` with botlib (code duplication/stub adapter)
- **Runtime: `botlib/be_aas_file.c`** — reads and validates `.aas` files at bot initialization; deserializes all 14 lumps into memory
- **Consumers of structs:**
  - `be_aas_reach.c` — builds `aas_reachability_t` links during reachability computation
  - `be_aas_route.c` — traverses reachability graph and caches paths; uses `aas_areasettings_t` reachability index
  - `be_aas_move.c` — executes movement for each travel type; queries `aas_area_t` and `aas_bbox_t`
  - `be_aas_sample.c` — maps points → areas; queries `aas_node_t` BSP tree
  - `be_aas_cluster.c` — manages portal connectivity; organizes areas into `aas_cluster_t` groups
  - `be_aas_entity.c` — links dynamic entities to reachability areas
- **All AAS constants** (travel types, presence types, face flags, area contents) are used throughout botlib; also referenced in `game/` VM for entity properties and in server collision queries

### Outgoing (what this file depends on)

- **`vec3_t`** — from `q_shared.h` (including translation unit must define it)
- **No explicit runtime dependencies** — this is a pure format definition consumed by other modules that include it

## Design Patterns & Rationale

**1. Versioned Binary Format with Lump Directory**
- Magic ID + two version constants (`AASVERSION_OLD=4`, `AASVERSION=5`) enable backward compatibility and format evolution
- 14-lump layout (offset + length pairs in header) is a classic id-tech pattern: single monolithic file with indexed sections for fast random access
- Allows deserialization to skip sections or detect incompatible data without full parse

**2. Travel Type Vocabulary**
- 19 movement modes (TRAVEL_WALK through TRAVEL_FUNCBOB) are not just physics states but **capability descriptors** — each encodes a distinct bot behavior (crouch, jump variants, special techniques like rocket/BFG/grapple, teleport types, elevator)
- Additional flags (`TRAVELFLAG_NOTTEAM1`, `TRAVELFLAG_NOTTEAM2`) allow per-reachability team restrictions
- This design allows the routing algorithm to prefer certain movement types and enforce game-specific restrictions without hard-coded logic

**3. Presence-Type Collision Classification**
- Three discrete presence types (NONE, NORMAL, CROUCH) enable separate bounding boxes per area for each body size
- Avoids costly capsule-sweep traces at runtime; instead, bot queries precomputed `aas_bbox_t` for its size class
- Enables tight areas (e.g., vents) to exclude large bots but allow small bots

**4. Reachability as Directed Graph Edges**
- `aas_reachability_t` is the core navigation primitive: areanum, travel type, start/end points, travel time
- Start/end points are **essential** for movement prediction (`be_aas_move.c`): enables jump-arc validation, ladder climbing, water/air transitions
- Travel time is precomputed distance metric; enables fast pathfinding without re-simulating movement

**5. Cluster/Portal Hierarchy**
- Two-level organization: areas within clusters, clusters connected by portals
- Allows hierarchical routing: Dijkstra within cluster, then inter-cluster via portal connectivity
- Portal areas themselves are areas (not abstract), enabling PVS/content queries on cluster boundaries

**6. Compact Bitfield Packing**
- `AREACONTENTS_MODELNUMSHIFT` / `AREACONTENTS_MAXMODELNUM`: packs up to 255 moving models into one int field (bits 24–31)
- `TRAVELTYPE_MASK` / `TRAVELFLAG_*`: separates travel-type enum (bits 0–23) from flags (bits 24–31)
- Minimizes per-area struct overhead; trades bit arithmetic for memory efficiency (important in 2005 on low-RAM systems)

## Data Flow Through This File

**Offline (Map Compile)**
```
Map (BSP) → bspc/map_q3.c → bspc/aas_create.c (build geometry)
         → bspc/aas_reach.c (compute reachability)
         → bspc/aas_cluster.c (identify clusters/portals)
         → bspc/aas_store.c (linearize to lump layout)
         → AAS_WriteAASFile() → .aas binary file (14 lumps)
```

**Runtime (Bot Navigation)**
```
Bot init: AAS_LoadAASFile() → deserialize 14 lumps → aasworld singleton
          (aas_header_t points to all lump offsets/lengths)

Per frame: Bot queries:
  - AAS_PointAreaNum(pos) → area index via aas_node_t BSP tree walk
  - AAS_AreaRouteToGoalArea(from, to) → traverse aas_reachability_t chain
  - AAS_PredictClientMovement(start, travel_type) → simulate via aas_bbox_t bounds
```

**State Transitions in Reachability Graph**
- Each `aas_reachability_t` is a directed edge; edges form the navigation graph
- Path = sequence of reachability links, each with its own travel type and time cost
- Routing cache (`be_aas_route.c`) memoizes Dijkstra results per (source area, destination area) pair

## Learning Notes

**What a developer studying this engine learns:**

1. **Spatial Subdivision Strategy (Early 2000s)**: Hierarchical areas + clusters beats flat space partitioning for fast multi-mode pathfinding. Modern engines use navmeshes or grid graphs; Q3 predates those mainstream approaches.

2. **Movement Capability Encoding**: Instead of "can the bot reach X?", the format asks "what movement modes can get the bot from A to B?" This inversion of control is elegant: bots don't query a generic pathfinder; they follow a pre-computed map of **what their body can do**.

3. **Presence-Type Collision**: The 3-level system (PRESENCE_NONE, NORMAL, CROUCH) is a low-overhead alternative to continuous-space collision. Still used in some commercial engines for performance-critical bot AI.

4. **Binary Format Discipline**: The lump-based layout is almost serializable-without-endian-handling (each lump is an array of fixed-size structs). The `AAS_SwapAASData` function likely converts between big-endian (map compile server) and little-endian (client execution). This was standard practice for shipping cross-platform games in 2005.

5. **Reachability as First-Class Citizen**: Modern pathfinders often treat edges as implicit (query a function). Here, reachability **is stored**, allowing per-edge metadata (start/end point, travel time) and team-specific restrictions. Very data-driven.

6. **Idiomatic Difference from Modern Engines**: 
   - Modern navmeshes are continuous (float coordinates); AAS areas are discrete (int indices).
   - Modern graphs are often 4- or 8-connected grids; AAS is explicitly authored (designer or compiler decides which areas connect).
   - No runtime graph construction; format is "final" at load time. Modern engines often validate and rebuild graphs at runtime.

## Potential Issues

**1. Format Versioning Compatibility**
- Two version constants defined (`AASVERSION_OLD=4`, `AASVERSION=5`), implying a past format change. Runtime code must handle both; if old maps are loaded with code expecting version 5, deserialization could fail or corrupt data.

**2. Endianness & Portability**
- Structs contain `int`, `float`, `vec3_t` arrays; binary layout is platform-dependent. `AAS_SwapAASData` is essential but easy to forget or get wrong on new platforms. No compile-time format validation (e.g., `static_assert(sizeof(aas_reachability_t) == X)`) to catch struct padding issues.

**3. Hardcoded Limits**
- `MAX_TRAVELTYPES = 32` but only 19 are defined. If future content needs more travel types, this constant must be increased and `.aas` files regenerated.
- `AREACONTENTS_MAXMODELNUM = 0xFF` (255 models max per area). Large maps with many movers could hit this.

**4. No Type Safety in Indices**
- Area numbers, face indices, edge indices, node indices are all `int`. No typedef wrapper to prevent accidental mixing (e.g., passing a face index where an area index is expected). Modern C would use `typedef int areanum_t` for clarity.
