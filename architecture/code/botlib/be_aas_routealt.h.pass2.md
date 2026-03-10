# code/botlib/be_aas_routealt.h — Enhanced Analysis

## Architectural Role

Alternative routing is a tactical pathfinding augmentation layered atop the core AAS system. While `be_aas_route.c` computes the single optimal path between areas, `be_aas_routealt.c` provides multiple candidate waypoints to enable bot AI to select tactically diverse routes. This separation allows the game VM's bot FSM (likely in `code/game/ai_dmq3.c` or `code/game/ai_dmnet.c`) to avoid predictable routes during goal selection, improving bot behavior against player pattern-matching.

## Key Cross-References

### Incoming (who depends on this file)
- Game VM bot AI goal selection (`code/game/ai_dmq3.c` via `trap_BotLibAlternativeRouteGoals` syscall)
- AAS lifecycle management in `code/botlib/be_aas_main.c` calls `AAS_InitAlternativeRouting` and `AAS_ShutdownAlternativeRouting` during setup and teardown
- The `type` parameter in `AAS_AlternativeRouteGoals` suggests game VM passes tactical strategy hints from its FSM state

### Outgoing (what this file depends on)
- Uses AAS cluster traversal via internal `AAS_AltRoutingFloodCluster_r` (defined in `.c` file)
- Depends on shared global `aasworld` singleton initialized by `be_aas_main.c`
- References reachability and area connectivity data computed by `be_aas_reach.c` and `be_aas_cluster.c`
- Works within the same travel-type bitmask system as `be_aas_route.c` and `be_aas_move.c`

## Design Patterns & Rationale

**Modular decoupling:** By isolating alternative routing behind a public API, botlib keeps the core Dijkstra-based routing (`be_aas_route.c`) lean, minimizing performance impact for simpler queries.

**Strategy type parameter:** The `type` argument allows different tactical modes (e.g., "find flanking routes," "high-ground alternatives") without multiplying function overloads — a common pattern in era-appropriate C library design.

**Lifecycle symmetry:** The `AASINTERN`-gated init/shutdown pair ensures the subsystem is initialized only once during `AAS_LoadMap` and cleaned up on `AAS_Shutdown`, preventing double-init bugs.

## Data Flow Through This File

1. **Input:** Bot AI calls `AAS_AlternativeRouteGoals` with start/goal positions (world-space), area indices, travel constraints (walk/swim/jump/etc.), and a strategy type.
2. **Transformation:** Internal cluster-flood algorithm (similar to main routing but exploring multiple branches instead of greedy selection) identifies candidate waypoint areas meeting the constraints.
3. **Output:** Returns an array of `aas_altroutegoal_t` structures (goal candidates) and count; bot's decision-making layer picks the best fit based on current tactical situation and AI personality.
4. **State:** Likely maintains cached candidate sets per (start, goal, type) triplet to amortize cost across multiple bot decision cycles.

## Learning Notes

- **Tactical AI complexity:** Q3A bots don't just pathfind; they reason about route diversity to avoid player exploitation. This is qualitatively different from modern shortest-path GIS algorithms.
- **Modular AAS architecture:** The subsystem breakdown (reach → route → routealt → entity queries) mirrors a layered reasoning system: "what's reachable?" → "what's the best path?" → "what are alternatives?" → "where's the player?"
- **Era-appropriate abstraction:** A modern engine might express tactical options via a behavior tree or GOAP planner; Q3A uses a procedural cluster flood with strategy hints. Both are valid; this scales well to a few hundred bot AI frames/sec on 2000s hardware.
- **Implicit caching hints:** The `type` parameter and separate subsystem suggest routing alternatives are computationally non-trivial and warrant caching; the separation allows botlib to manage cache lifecycle independently.

## Potential Issues

None directly inferable from the header. The `type` parameter uses `int` rather than an `enum`, which is minor style (era-typical C89). No obvious NULL-safety issues: `altroutegoals` is caller-allocated, and `maxaltroutegoals` provides bounds. Internal LRU eviction in the routing cache (`AAS_FreeOldestCache` in `be_aas_route.c`) may affect alternative goal freshness under memory pressure, but this is coordinated at the botlib level.
