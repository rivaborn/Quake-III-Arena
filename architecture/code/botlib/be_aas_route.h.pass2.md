# code/botlib/be_aas_route.h — Enhanced Analysis

## Architectural Role

This file exposes the **hierarchical pathfinding and caching layer** that underpins bot navigation in multiplayer matches with up to 32 bots. It bridges two distinct worlds: the **offline AAS compiler** (`code/bspc`) which precomputes reachability graphs and clusters, and the **runtime bot FSM** (`code/game/ai_dmnet.c`) which repeatedly queries travel times with per-frame latency budgets. The routing subsystem implements a multi-layered LRU cache strategy atop cluster-hierarchy pathfinding, making repeated queries O(1) while transparently handling dynamic area enable/disable events (doors, moving brushes). Unlike modern navmesh systems, this design separates precomputation (reachability) from runtime (caching), allowing 32 bots to pathfind without stalling the frame loop.

## Key Cross-References

### Incoming (who depends on this file)

**Server-side bot AI** (`code/game/`):
- `ai_dmnet.c` / `ai_move.c` / `ai_goal.c` invoke `AAS_*` functions via `trap_BotLib*` syscalls (opcodes 200–599)
- Each bot calls `AAS_AreaTravelTimeToGoalArea()` during goal re-evaluation and movement planning

**botlib internal modules**:
- `be_ai_move.c`, `be_ai_goal.c`, `be_ai_weap.c` call routing functions directly (no syscall overhead)
- `be_aas_main.c` lifecycle: `AAS_InitRouting()` on map load, `AAS_FreeRoutingCaches()` on unload
- `be_aas_routealt.c` (alternate routes) depends on routing infrastructure for fallback pathfinding

**Supporting subsystems**:
- `be_aas_entity.c` invalidates cached routes when entities (doors, platforms) move
- `be_aas_cluster.c` provides the hierarchical traversal structure underlying efficient routing

### Outgoing (what this file depends on)

**Internal implementations** (same `.c` file):
- `AAS_GetAreaRoutingCache()`, `AAS_UpdateAreaRoutingCache()` — cache lookup and LRU eviction
- `AAS_NearestHideArea()` — specialized pathfinding for evasion logic
- `AAS_FreeOldestCache()` — explicit FIFO cache eviction when memory budget exceeded

**Lower-layer subsystems**:
- **Sampling** (`be_aas_sample.h`): `AAS_PointAreaNum()`, `AAS_TraceClientBBox()` to locate entities and trace paths
- **Reachability** (`be_aas_reach.h`): `AAS_NextAreaReachability()`, `AAS_ReachabilityFromNum()` to enumerate travel links
- **BSP/Collision** (`be_aas_bspq3.c`): `AAS_Trace()`, `AAS_EntityCollision()` for movement validation
- **Entity tracking** (`be_aas_entity.h`): `AAS_UpdateEntity()`, `AAS_ResetEntityLinks()` for dynamic obstacle handling
- **Cluster hierarchy** (`be_aas_cluster.h`): Portal-tree structure for hierarchical pathfinding

**Memory subsystem** (`l_memory.h`): Allocation/freeing of large cache arrays (potentially megabytes per map).

## Design Patterns & Rationale

**1. Hierarchical + Cache Hybrid**  
The routing doesn't recompute shortest-paths per query. Instead it:
- **First load**: AAS compiler pre-computes all inter-area reachabilities offline
- **First query**: Dijkstra via cluster hierarchy (portal-to-portal pathfinding)
- **Subsequent queries**: O(1) LRU cache lookup; miss triggers cache rebuild

This amortizes expensive pathfinding across repeated similar queries (e.g., "how to reach rocket launcher?").

**2. Travel-Type Polymorphism**  
`AAS_TravelFlagForType(traveltype)` converts semantics into bitmasks. Bots can ask:
- "Reachable by walking?" (`TFL_WALK`)
- "Reachable by jumping?" (`TFL_JUMP`)
- "Reachable by rocket-jumping?" (`TFL_ROCKETJUMP`)
- "Reachable without water?" (invert `TFL_WATER`)

This is **content-aware** pathfinding, not uniform-cost. Different travel types have different reachability graphs.

**3. Staged Commitment (Lookahead Validation)**  
`AAS_PredictRoute(..., stopevent, stopcontents, ...)` **simulates** movement along a path before commitment:
```c
if (AAS_PredictRoute(&route, currentArea, botOrigin, 
                     lavaLaucherArea, TFL_WALK, 
                     32, 2000, EVENT_LAVA_SPLASH, 
                     CONTENTS_LAVA, 0, 0))
  // Path triggers hazard; choose alternate
else
  // Safe to travel
```
This is **not** a pathfinder—it's a validator that checks for stop-events. Modern engines use navmesh raycasts; here it's explicit physics simulation.

**4. Dynamic Area Disabling**  
`AAS_EnableRoutingArea(areanum, 0)` blocks an area at runtime. The caching layer must **implicitly invalidate** routes that touch that area. This requires either:
- Explicit cache flush calls (not visible in header), or
- Lazy invalidation (rebuild on next miss)

The presence of `AAS_FreeOldestCache()` in cross-refs suggests time-based eviction + eventual cache turnover rather than explicit invalidation.

## Data Flow Through This File

**Initialization** (called once per map load):
```
Server starts level
  → botlib/be_aas_main.c : AAS_InitRouting()
    ├─ Allocates routing cache arrays (travel-time lookup tables)
    ├─ Precomputes or loads cached cluster hierarchy
    └─ Initializes entity-link heap for dynamic object tracking
```

**Per-Frame Bot Movement** (called once per bot per frame):
```
Game VM (server)
  → ai_dmnet.c : AINode_Stand() / AINode_Combat() / etc.
    → ai_goal.c : BotChooseGoal()
      → AAS_RandomGoalArea() [pick random reachable area]
      → AAS_AreaTravelTimeToGoalArea(myArea, myOrigin, goalArea, TFL_WALK)
        → Cache hit: O(1) return cached travel time
        → Cache miss: Dijkstra via cluster tree, populate cache
    
    → ai_move.c : BotMovementAI()
      → AAS_PredictRoute(&route, myArea, myOrigin, 
                         nextReachArea, TFL_WALK, 
                         8, 1000, EVENT_LAVA, CONTENTS_LAVA, 0, 0)
        → Simulate physics along planned segment
        → If hit lava, return non-zero; bot reconsiders route
        → Else return 0; bot commits to movement
```

**Cache Invalidation** (when level geometry changes):
```
Game event: Door opens → blocking area unblocked
  → AAS_EnableRoutingArea(doorArea, 1)
    → Modifies area routing state
    → Routes touching doorArea become stale
    → Next query cache misses, rebuilds as needed
```

## Learning Notes

1. **Precomputation Amortization in Practice**: The AAS file represents several seconds of offline work (reachability computation, cluster construction) enabling immediate O(1) queries at runtime. Modern engines (Unreal, Unity) use similar patterns with navmesh + pathfinding caches.

2. **Multi-Modal Movement**: Unlike human pathfinding (uniform graph), bots have 15+ travel modes: walk, jump, step-over, swim, ladder-climb, teleport, jump-pad, rocket-jump, double-jump, etc. The `TravelFlagForType()` pattern encodes this polymorphism into bitmasks—each mode defines which areas are reachable. This is **more expressive than a single cost metric**.

3. **Lookahead as Safety**: `AAS_PredictRoute()` is a **bounded simulation**, not a pathfinder. By simulating the next 8 areas up to 2 seconds forward, bots detect traps (e.g., rocket launcher on a lava island) before walking into them. This is a pragmatic middle ground between full AI planning and greedy nearest-goal.

4. **Memory-CPU Tradeoff at Scale**: With 32 bots per server, per-frame pathfinding would cost ~32 × O(areas log areas). LRU caching + hierarchical clustering reduce this to ~32 × O(1) on average, enabling frame-rate stability. Cache misses (goal changes, teleports) amortize to negligible cost.

5. **Idiomatic to Q3 Era**: Explicit precomputation + query-caching reflects mid-2000s constraints (single-threaded frame loop, limited CPU). Modern engines use lock-free hierarchical search (e.g., HPA*, theta*) instead.

## Potential Issues

1. **Cache Invalidation Opaqueness**: Header declares `AAS_EnableRoutingArea()` but doesn't specify cache invalidation semantics. Looking at `.c` implementation (not visible here) is necessary to understand latency bounds after area state changes.

2. **Return-Type Ambiguity**: `AAS_AreaTravelTime()` returns `unsigned short` (0–65535). Does `0` mean "unreachable" or "same location"? Cross-ref shows this is queried frequently; an off-by-one interpretation could cause bot navigation bugs.

3. **Route Prediction Limits**: `AAS_PredictRoute(..., maxareas=8, maxtime=2000, ...)` caps lookahead. What happens if the next reachable area is 9+ steps away? Header doesn't clarify fallback behavior—does bot cancel movement, or does it optimistically proceed?

4. **Concurrency Not Visible**: Multiple bots querying overlapping cached routes simultaneously—no locking primitives exposed in header. Either (a) cache is thread-safe internally, or (b) botlib is always called single-threaded per frame (likely, given Q3's architecture).

5. **Travel-Type Explosion**: 15+ travel types × bitmask combinations could generate matrix-like complexity in cache keys. No visible documentation of which combinations are actually used, making it hard to reason about cache efficiency.
