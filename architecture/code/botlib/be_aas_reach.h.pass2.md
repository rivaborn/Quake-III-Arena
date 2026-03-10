# code/botlib/be_aas_reach.h — Enhanced Analysis

## Architectural Role

This header sits at the heart of botlib's **Area Awareness System (AAS)** initialization and query layer. It bridges the gap between AAS world construction (map load phase, handled privately by `#ifdef AASINTERN` functions) and runtime bot decision-making (public query API). During map load, it drives expensive reachability computation across frames (`AAS_InitReachability` + `AAS_ContinueInitReachability`); at runtime, it provides the spatial and environmental queries that feed goal selection (`be_ai_goal.c`) and movement validation (`be_aas_move.c`). The reachability graph computed here is the backbone of bot pathfinding: `be_aas_route.c` later consults these pre-computed edges to find optimal paths.

## Key Cross-References

### Incoming (who depends on this file)
- **be_aas_route.c**: Relies on `AAS_NextModelReachability` to iterate reachabilities and build routing caches; queries area properties to validate travel legality
- **be_ai_goal.c**: Calls `AAS_BestReachableArea` to locate optimal goal destinations; filters candidates with `AAS_Area{Liquid,Lava,Slime,DoNotEnter,Crouch}` predicates
- **be_ai_move.c**: Queries `AAS_AreaGrounded`, `AAS_AreaLadder`, `AAS_AreaSwim` to classify movement constraints before executing travel
- **be_aas_move.c**: Uses reachability metadata to validate jump arcs and predict movement feasibility
- **be_aas_sample.c**: Links entities to areas; cross-checks area properties during spatial queries

### Outgoing (what this file depends on)
- **be_aas_reach.c** (implementation): Owns the reachability computation; manipulates the global `aasworld` singleton
- **be_aas_def.h** (types): `aas_link_t` structure for area connectivity lists
- **be_aas_main.c**: Shared initialization infrastructure, error handling, string/model index translation
- **q_shared.h**: `vec3_t` for spatial coordinates

## Design Patterns & Rationale

**Lazy/Incremental Initialization Pattern**: The split between `AAS_InitReachability` (kick-off) and `AAS_ContinueInitReachability(float time)` amortizes reachability computation across multiple server frames, preventing the 1–2 second hitches that would occur on large maps if computed synchronously. Each call gets a time budget; the return value signals completion. This pattern is essential in 2000s game engines where frame budgets were tight and maps could be large.

**Internal/Public API Boundary**: The `#ifdef AASINTERN` guard hermetically seals initialization (`AAS_InitReachability`, `AAS_ContinueInitReachability`, `AAS_BestReachableLinkArea`) from external callers. Only query functions are exposed publicly. This enforces the invariant that reachability graphs are immutable at runtime—a critical assumption for deterministic pathfinding.

**Spatial Query Triads**: Functions like `AAS_BestReachableArea(origin, mins, maxs, *goalorigin_out)` and `AAS_BestReachableFromJumpPadArea(origin, mins, maxs)` follow a consistent pattern: accept a bounding box (not a point), return an area index and optionally write a concrete position. This supports bot goal selection where a destination must be within a reachable area *and* have a valid position inside that area (e.g., not inside a wall).

**Predicate Filtering**: The eleven boolean queries (`AAS_AreaCrouch`, `AAS_AreaLava`, etc.) are pure, side-effect-free classifiers. Together they form a vocabulary for behavioral constraints that bot AI applies when filtering candidate goals or validating movement.

## Data Flow Through This File

1. **Map Load → Async Computation**:  
   `AAS_LoadMap` → `AAS_InitReachability()` stores initialization state; each frame `AAS_ContinueInitReachability(deltaTime)` incrementally computes travel edges between areas (14+ travel types: walk, jump, swim, climb ladder, teleport, jump pad, etc.), storing them in `aasworld.reachability`.

2. **Runtime Goal Selection**:  
   `be_ai_goal.c` calls `AAS_BestReachableArea(bot_pos, mins, maxs, &goal_pos)` → returns nearest/best reachable area and a concrete position within it → goal is set.

3. **Routing & Movement**:  
   `be_aas_route.c` iterates reachabilities using `AAS_NextModelReachability` and area predicates to validate candidate next-steps; `be_aas_move.c` simulates physics to confirm jump/step/swim feasibility, consulting `AAS_AreaLiquid`, `AAS_AreaLadder` for constraints.

4. **Special Cases**:  
   Jump pad areas are queried with `AAS_BestReachableFromJumpPadArea(origin, mins, maxs)` to find pads that can launch bots toward a goal—enabling bots to exploit map tricks.

## Learning Notes

Studying this file reveals:
- **Environmental richness**: Quake III distinguishes lava, slime, and generic water—each has different gameplay semantics (damage, movement rules).
- **Travel-type taxonomy**: The system abstracts 14+ distinct movement modes (walk, jump, ladder climb, swim, teleport, jump pad, elevator, ramp, barrier, water jump, walk-off-ledge, func_bobbing). Modern engines often use animation state machines; Q3 uses explicit travel types for deterministic bot logic.
- **Bounding-box reasoning**: Spatial queries use 3D boxes, not points—critical for ensuring goals are physically reachable by a bot's collision hull, not just by a raycast.
- **Model-based dynamics**: `AAS_NextModelReachability` acknowledges that moving BSP models (doors, platforms) create dynamic reachability edges—bot pathfinding must refresh these per-frame.

**Era-specific idioms**: Lazy initialization with time budgets was standard practice in early-2000s engines; modern engines often precompute everything offline. The separation of `_init` (internal) and query APIs is a predecessor to modern plugin/module boundary conventions.

## Potential Issues

- **Model reachability staleness**: If a mover (door, lift) changes position, the reachability edges it creates must be invalidated and recomputed. If the update cycle drifts out of sync with bot pathfinding queries, bots could request paths through stale edges (e.g., a closed door).
- **Jump-arc validation**: The caller relies on `AAS_ContinueInitReachability` to validate that computed jumps are physically feasible given gravity and client friction cvars. If these cvars change mid-game without recomputing reachability, bots may attempt invalid jumps.
- **Area classification ambiguity**: An area could theoretically satisfy multiple predicates (e.g., both crouch-only *and* has a ladder). The calling code must handle precedence (e.g., prefer ladder travel if available).
