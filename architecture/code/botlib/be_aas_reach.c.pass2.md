# code/botlib/be_aas_reach.c — Enhanced Analysis

## Architectural Role

This file is the **geometry-to-graph converter** of the botlib subsystem. It transforms raw BSP-derived AAS areas into a navigable **directed graph** by discovering all feasible movement transitions between adjacent areas. It sits between spatial sampling (`be_aas_sample.c`) and path routing (`be_aas_route.c`), acting as a critical compile-time (per-map-load) preprocessing phase. The output—`aasworld.reachability[]` array—becomes the edge list for Dijkstra-based pathfinding.

## Key Cross-References

### Incoming (who depends on this file)
- **`be_aas_main.c`**: Calls `AAS_InitReachability()` once per map load as part of the AAS initialization pipeline (`AAS_LoadMap()` → `AAS_Setup()` → `AAS_InitReachability()`).
- **Per-frame game loop** (via `be_interface.c`): Calls `AAS_ContinueInitReachability(time)` each frame until completion; gates navigation readiness on `aasworld.numreachabilityareas == aasworld.numareas + 1`.
- **`be_aas_route.c`**: Consumes finalized `aasworld.reachability[]` array and per-area `areasettings[].firstreachablearea` indices to build routing caches; never calls reach functions directly.
- **`be_aas_debug.c`**: Calls `AAS_ShowReachability()` to visualize computed links for offline debugging.

### Outgoing (what this file depends on)
- **`be_aas_sample.c`**: Calls `AAS_TraceClientBBox()`, `AAS_AreaVolume()`, `AAS_PointAreaNum()`, `AAS_PointContents()`, `AAS_LinkEntityClientBBox()`, `AAS_UnlinkFromAreas()`, `AAS_TraceAreas()`, `AAS_PointInsideFace()` — all geometric queries.
- **`be_aas_move.c`**: Calls `AAS_PredictClientMovement()`, `AAS_HorizontalVelocityForJump()`, `AAS_RocketJumpZVelocity()`, `AAS_BFGJumpZVelocity()`, `AAS_DropToFloor()`, `AAS_OnGround()`, `AAS_Swimming()` — movement simulation for jump/fall validation.
- **`be_aas_bspq3.c`**: Calls `AAS_NextBSPEntity()`, `AAS_ValueForBSPEpairKey()`, `AAS_VectorForBSPEpairKey()`, `AAS_FloatForBSPEpairKey()`, `AAS_BSPModelMinsMaxsOrigin()` — entity key-value parsing for teleporters, elevators, jump pads.
- **`be_aas_cluster.c`**: Indirectly; reachability links store `TRAVEL_*` types used by cluster portal routing logic.
- **Global `aasworld`**: Reads/writes the master AAS data structure; allocates `aasworld.reachability[]` in `AAS_StoreReachability()`.

## Design Patterns & Rationale

### Visitor Pattern with Type Dispatch
Each `AAS_Reachability_*()` function encapsulates the geometric/physics logic for one movement class (swim, walk, jump, ladder, etc.). The main loop in `AAS_ContinueInitReachability()` is a **simple dispatcher** that calls each visitor on every pair of adjacent areas in priority order. This is elegant because:
- New movement types can be added without modifying the dispatcher
- Each classifier is self-contained and testable
- The order (geometric before entity-driven) reflects computational cost

### Linked-List-to-Array Conversion (Memory Tradeoff)
- **During computation**: Use per-area linked lists of `aas_lreachability_t` (low allocation overhead, cache-unfriendly during iteration).
- **After completion**: Convert to flat `aasworld.reachability[]` array indexed by `areasettings[area].firstreachablearea` (high iteration throughput, compact memory, cache-friendly for pathfinding).

This is **classic space-time optimization**:
- Computation phase favors insertions (linked list `O(1)` prepend).
- Query phase favors sequential access (array iteration with tight loop).

### Incremental Startup via Frame Budget
`AAS_ContinueInitReachability()` processes only `REACHABILITYAREASPERCYCLE` (15) areas per call. This **spreads computation across frames** to prevent startup stalls. Visible in:
- `Sys_MilliSeconds()` time tracking (though not enforced as hard deadline).
- Per-frame loop in main engine or bot server loop.
- Essential for **online multiplayer** where server startup latency affects join times.

### Fixed Heap Pool with Graceful Overflow
`AAS_MAX_REACHABILITYSIZE = 65536` is a pre-allocated pool. If exhausted:
- `AAS_AllocReachability()` returns `NULL`.
- `AAS_Error()` is called once, then silent degradation (no more links added).
- **Game remains playable** with incomplete navigation graph (bots navigate best-effort on available links).

This is pragmatic for a 2000s engine with predictable worst-case memory usage and no dynamic heap fragmentation.

## Data Flow Through This File

1. **Initialization** (`AAS_InitReachability`):
   - Allocate heap pool and per-area linked-list heads.
   - Call `AAS_SetWeaponJumpAreaFlags()` to mark high-value item areas.
   - Set `aasworld.numreachabilityareas = 1` (area 0 excluded; algorithm starts at area 1).

2. **Per-Frame Iteration** (`AAS_ContinueInitReachability`):
   - For areas `numreachabilityareas` to `min(numreachabilityareas + REACHABILITYAREASPERCYCLE, numareas)`:
     - Against all adjacent areas, call each `AAS_Reachability_*()` classifier.
     - If classifier returns `qtrue`, link is added to `areareachability[area]` head.
     - Increment counters (`reach_swim++`, etc.).
   - On final iteration (area index > numareas), call entity-driven passes (teleport, elevator, jump pad, func_bobbing).
   - Call `AAS_StoreReachability()`.

3. **Finalization** (`AAS_StoreReachability`):
   - Flatten `areareachability[area]` linked lists into a single `aasworld.reachability[]` array.
   - Write `areasettings[area].firstreachablearea` and `.numreachableareas` for each area.
   - Free heap and `areareachability[]`.

4. **Runtime Query** (not in this file):
   - `be_aas_route.c` iterates `aasworld.reachability[areasettings[area].firstreachablearea...]` to build routing graphs.

## Learning Notes

### Idiomatic Q3 AAS Design (vs. Modern Approaches)
- **Hand-rolled type detection**: Each movement type (swim, jump, ladder) has explicit geometric predicates. Modern navmesh systems (`Recast/Detour`, `NavMesh Pro`) are data-driven and learned from painted surfaces.
- **Direct geometry queries**: Reachability relies on swept-box traces and BSP collision queries. No precomputed distance fields or visibility.
- **Entity coupling**: Teleporters/elevators/jump pads are detected via BSP key-value pairs, binding AAS to level design convention. Modern systems decouple navigation from specific entity types.
- **Travel-type classification**: Each link carries a `traveltype` (enum) that doubles as both a movement constraint and a cost hint. Modern systems use cost maps with per-traversal-type multipliers.

### Frame-Time Awareness
This is exemplary of **pragmatic 2000s game programming**:
- Assumes 60Hz+ frame rate during compute phase.
- Spreads work to prevent stalls visible to players (especially in online multiplayer).
- No sophisticated scheduling; just `REACHABILITYAREASPERCYCLE` as a hardcoded budget.
- Modern engines might use work queues with adaptive per-frame time budgets.

### Direction and Asymmetry
Reachability is **directed**: you can jump down but may not jump back up. The pathfinder must handle both directions independently. Modern navmesh systems often enforce bidirectionality for simplicity, at the cost of potentially invalid reverse paths.

### Weapon Jump & High-Value Items
The `AREA_WEAPONJUMP` flag marks areas reachable via rocket/BFG jumps, which are expensive but high-value for item collection. This **game-logic-aware optimization** embeds strategic gameplay knowledge (weapon jumps are special, risky, but valuable) into the navigation graph. Few modern engines expose this level of gameplay awareness to pathfinding.

## Potential Issues

1. **Heap Exhaustion Without Visibility**: If `AAS_MAX_REACHABILITYSIZE` is hit, the map's navigation graph becomes incomplete, but the game continues silently. A very complex map (thousands of areas) could have subtly broken pathfinding without obvious error symptoms.

2. **Jump Validation via Discrete Simulation**: `AAS_PredictClientMovement()` uses discrete 10ms time steps. A sufficiently tight jump might be missed (false negative) or a broken jump might pass (false positive) due to step aliasing.

3. **Entity Key-Value Dependency**: Teleporter/elevator reachabilities depend on properly-keyed BSP entities (correct `target`/`targetname` pairs, presence of `speed` key). Mapper errors silently break reachability without warnings.

4. **Frame-Rate Sensitivity**: The incremental computation assumes steady frame rate. On highly variable frame rates (framerate dips), `AAS_ContinueInitReachability()` might take much longer to finish (though this is rarely an issue in practice).

5. **Grapple Hook Opt-Out**: The `calcgrapplereach` global flag can disable grapple link computation. If set incorrectly (e.g., by a mod or config error), grapple-based bots would have incomplete navigation without obvious failure mode.

---

*This file exemplifies the blend of pragmatic CPU/memory management and deep gameplay knowledge that characterizes Q3A's architecture. The incremental startup, fixed-size pools, and game-logic-aware travel types would benefit from modern profiling and potentially adaptive scheduling, but the core geometry-to-graph pipeline is sound.*
