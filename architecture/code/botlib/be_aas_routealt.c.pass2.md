# code/botlib/be_aas_routealt.c — Enhanced Analysis

## Architectural Role
This file extends the core **AAS routing subsystem** (be_aas_route.c) with a specialized **alternative path discovery** capability that identifies diverse waypoint routes between start and goal positions. While be_aas_route.c implements shortest-path Dijkstra queries, be_aas_routealt.c solves a different problem: finding mid-range "flanking" waypoints that allow bots to take tactically interesting non-direct paths without traveling dramatically longer distances. It integrates tightly with be_aas_route.c (reuses `AAS_AreaTravelTimeToGoalArea`), be_aas_sample.c (area geometry), and be_aas_reach.c (connectivity), forming part of botlib's multi-layered path-planning strategy.

## Key Cross-References

### Incoming (Callers)
- **code/game/g_bot.c** or **code/server/sv_bot.c** — Bot AI frame logic or bot initialization likely calls `AAS_AlternativeRouteGoals` to generate waypoint options for evasion/flanking behaviors (inferred from botlib's role in the game VM boundary)
- **botlib higher-level AI modules** (be_ai_move.c, be_ai_goal.c) — Movement FSM or goal selection likely queries alternative routes when deciding between direct vs. tactical paths

### Outgoing (Dependencies)
- **be_aas_route.c** — `AAS_AreaTravelTimeToGoalArea` (queries cached/live Dijkstra routing between any two areas)
- **be_aas_reach.c** — `AAS_AreaReachability` (checks if an area has valid reachability links; gates eligible mid-range areas)
- **be_aas_sample.c** — Implicit via `aasworld.areas[].center` (accesses geometric center of each area for clustering)
- **be_aas_bspq3.c** (via aasworld) — `aasworld.faces` / `aasworld.faceindex` (reads navigation mesh connectivity to flood-fill clusters)
- **be_aas_debug.c** — `AAS_ShowAreaPolygons` (debug-only visualization in `#ifdef ALTROUTE_DEBUG` builds)
- **qcommon (q_shared.h)** — Vector macros, `Com_Memset`
- **botlib utilities** — `l_memory.h` (GetMemory/FreeMemory), `l_log.h` (Log_Write)

## Design Patterns & Rationale

### Conditional Compilation (Feature Toggle)
The entire module is guarded by `#ifdef ENABLE_ALTROUTING`. This is typical of **optional subsystems** in game engines: alternative routing can be disabled to reduce binary footprint or at map-load time if deemed unnecessary. Disabling it reduces all four functions to stubs returning 0/void, with zero runtime cost.

### Working Buffer Pattern (Scratch Space)
Three file-statics serve as **pre-allocated scratch buffers** shared across calls: `midrangeareas[]`, `clusterareas[]`, and `numclusterareas`. Instead of malloc-per-call, the module reuses these buffers, initialized once in `AAS_InitAlternativeRouting`. This is a **performance idiom** common in latency-critical game code—arena allocation via init/shutdown avoids frame-time fragmentation.

### Recursive Flood-Fill (Graph Traversal)
`AAS_AltRoutingFloodCluster_r` is a classic **recursive depth-first search** that consumes (invalidates) visited areas to partition the mid-range set into connected clusters. The algorithm is destructive: it clears `midrangeareas[areanum].valid` to prevent re-entry and double-counting. This works only because the buffer is re-zeroed each call to `AAS_AlternativeRouteGoals`.

### Spatial Clustering via Centroid Selection
Once a cluster is collected, the algorithm computes its geometric **centroid** (average of all area centers), then **greedily selects the single area closest to that centroid** as the representative waypoint. This heuristic balances **spatial diversity** (spread across the map) with **reachability guarantees** (staying within the mid-range set).

### Travel-Time Threshold Heuristics
The dual thresholds (`starttime ≤ 1.1 × direct` and `goaltime ≤ 0.8 × direct`) form a **probabilistically safe detour filter**:
- The 1.1× upper bound on start-time prevents areas too far from the player
- The 0.8× upper bound on goal-time ensures waypoints make progress toward the goal
- Together, they guarantee that routing via any mid-range area is never worse than ~1.9× the direct time (loose upper bound)

Rationale: Bots can only deviate meaningfully if the detour is "reasonable"—otherwise the behavior looks unintelligent.

## Data Flow Through This File

**Input:**
- Caller supplies: start/goal positions, start/goal area numbers, travel flags (`TRAVEL_*`), type filter (`ALTROUTEGOAL_*` bitmask)
- AAS world global (`aasworld`) provides: area centers, face connectivity, area settings (contents flags)

**Transformation:**
1. Measure direct route cost via `AAS_AreaTravelTimeToGoalArea(start, goal)`
2. Iterate all areas; for each, compute `starttime` and `goaltime` to candidate area
3. Filter by thresholds + type + reachability → populate `midrangeareas[]` with valid candidates
4. Partition valid areas into clusters via recursive flood-fill (marking each as visited)
5. For each cluster, compute centroid and select closest area as waypoint representative
6. Populate output `altroutegoals[]` with origin, area number, and travel-time deltas

**Output:**
- Array of `aas_altroutegoal_t`: one per cluster, ranked by discovery order
- Return count (0 if disabled, no valid candidates, or invalid input areas)

**Side Effects:**
- Clobbers `midrangeareas[]` and `clusterareas[]` (overwritten next call)
- Emits log entries to `l_log` for each candidate found (may spam if many mid-range areas exist)
- Calls debug visualization if `ALTROUTE_DEBUG` is defined

## Learning Notes

### Era-Appropriate Design Choices
This code reflects **early-2000s game engine conventions**:
- **Global scratch buffers** instead of heap allocation per-call (stack is scarce, malloc is slow)
- **Conditional compilation** for optional features (memory and CPU were tighter constraints)
- **Simple heuristics** (thresholds, centroid selection) instead of complex optimization (no machine learning, sparse search)
- **File-static globals** for module state, initialized/freed by explicit calls (no RAII, no automatic lifetime management)

### Contrast with Modern Engines
Modern engines would likely:
- Use stack allocations or a frame allocator to avoid globals
- Make alternative routing a query on-demand with no pre-allocated state
- Use a more sophisticated routing cache or decision tree (e.g., behavior tree, utility scoring)
- Parallelize clustering and selection

### AAS Navigation Hierarchy
This file reveals **three tiers of path planning**:
1. **Micro** (be_aas_move.c): Low-level movement simulation (gravity, friction, jumping)
2. **Macro** (be_aas_route.c): Shortest-path routing via Dijkstra on reachability graph
3. **Tactical** (be_aas_routealt.c): Diversity-seeking—find non-optimal but reasonable alternate routes for AI variety

This mirrors real game AI design: shortest path alone is deterministic and boring; alternative routing introduces tactical choice and replayability.

## Potential Issues

### Silent Integer Overflow
Travel times are stored as `unsigned short` (16-bit, ~65.5 second range). Areas with computed travel times exceeding 65535 will silently overflow, corrupting the mid-range threshold check. Mitigation: either cap input times or use wider types. Not a blocker for typical maps but a latent bug in edge cases.

### No Recursion Depth Limit
`AAS_AltRoutingFloodCluster_r` has no iteration limit. In pathological maps (huge open areas with dense area subdivision), recursion could overflow the stack. Mitigation: convert to explicit stack or add a max-depth guard. Unlikely in practice on Quake III Arena's designed maps, but risky for modded geometry.

### Unused `type` Filtering Logic
The conditional `if (!(type & ALTROUTEGOAL_ALL))` structure is verbose and redundant—the first clause `if (!(type & ALTROUTEGOAL_CLUSTERPORTALS ...))` already filters if neither bit is set. Could be simplified but does not affect correctness.
