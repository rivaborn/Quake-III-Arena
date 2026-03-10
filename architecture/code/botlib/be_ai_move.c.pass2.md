# code/botlib/be_ai_move.c — Enhanced Analysis

## Architectural Role

This file is the **movement execution engine** of the botlib pipeline, translating high-level pathfinding decisions into frame-by-frame physics-aware travel behaviors. It sits at a critical junction: above it, the game VM's AI FSM selects goals and calls `BotMoveToGoal`; below it, the AAS routing system provides path guidance and the EA (elementary action) layer translates results into bot `usercmd_t` input. Unlike reachability computation (offline, done in `be_aas_reach.c`) or routing (cached, done in `be_aas_route.c`), this module's job is **real-time per-frame execution**—it must handle partial failures, dynamic obstacles, stuck-bot detection, and graceful degradation with avoid-lists.

## Key Cross-References

### Incoming (who depends on this file)
- **code/game/g_bot.c** → calls `BotMoveToGoal`, `BotSetupMoveAI`, `BotShutdownMoveAI`, `BotAllocMoveState`, `BotFreeMoveState`, `BotInitMoveState` via the botlib import vtable during bot lifecycle and per-frame AI ticks
- **code/botlib/be_interface.c** → exposes public move AI symbols in `botlib_export_t` for the server

### Outgoing (what this file depends on)
- **code/botlib/be_aas_*.c** (reach, route, sample, entity, main): routing cost queries (`AAS_AreaTravelTimeToGoalArea`), area/reachability lookups, movement validation (`AAS_PredictClientMovement`), entity tracking
- **code/botlib/be_ea.c** → dispatches elementary actions (`EA_Move`, `EA_Jump`, `EA_Attack`, `EA_SelectWeapon`, `EA_View`, `EA_Crouch`, `EA_Command`) that accumulate into `bot_input_t`
- **libvars** (physics constants): `sv_gravity`, `sv_maxstep`, `sv_maxbarrier`, weapon indices, grapple mode flags
- **botimport** interface: debug printing, BSP entity queries, time access

## Design Patterns & Rationale

### 1. **Travel-Type Dispatch Pattern**
The `BotMoveToGoal` function branches based on `reach->traveltype` (walk, jump, ladder, grapple, etc.) to ~15 specialized handlers. This avoids a monolithic function and lets each travel type evolve independently. However, **dispatch happens every frame**, not once at reachability selection—allowing handlers to maintain internal sub-state (grapple arm time, jump-run-up counter, etc.).

### 2. **Per-Frame State Machine for Grapple & Weapon Jumps**
Functions like `BotTravel_Grapple`, `BotTravel_RocketJump` maintain multi-frame state via `grapplevisible_time`, `lastgrappledist`, and `reachability_time` in `bot_movestate_t`. This avoids awkward synchronization; each handler knows whether it's in the aim phase, fire phase, or recovery phase based on elapsed time. Timeout-based fallback (attempt → timeout → blacklist) is a pragmatic heuristic for getting unstuck.

### 3. **Two-Level Avoidance System**
- **Reachability blacklist** (`avoidreach[MAX_AVOIDREACH]`, `avoidreachtimes`, `avoidreachtries`): If a reachability fails N times within a window, it's temporarily blacklisted to force routing through alternate paths.
- **Avoid-spots** (`avoidspots[MAX_AVOIDSPOTS]`): Caller-supplied hazard zones (e.g., lava, enemy spawns) that are checked during reachability selection. This separation of concerns allows the AI FSM to dynamically inject tactical knowledge (e.g., "avoid this respawner").

### 4. **Fuzzy Area Finding (3×3×3 Grid)**
`BotFuzzyPointReachabilityArea` and `BotReachabilityArea` defensively sample a 3×3×3 grid of offset positions to handle floating-point boundary cases. This is slower than a single point query but far more robust—crucial for correctness when a bot is standing on a moving platform or mesh boundary.

### 5. **Early Ground Classification**
`BotReachabilityArea` does expensive classification (is this a mover? which area?) *once per frame* instead of every time reachability is queried. It caches the decision in the movement state and returns early for simple cases (walking on world → use fuzzy area).

## Data Flow Through This File

```
Per-frame input (from server snapshot):
  origin, velocity, viewoffset, entity/client num, move flags (on ground, teleported, etc.)
                            ↓
          BotInitMoveState(handle, initmove)
          Updates: ms->origin, ms->velocity, ms->viewangles, ms->moveflags
                            ↓
          BotMoveToGoal(result, movestate, goal, travelflags)
                       ├─→ In-air branch: finish travel (jump pad, etc.)
                       │    └─→ BotFinishTravel_*() handlers
                       └─→ On-ground branch: select next reach, execute travel
                            ├─→ BotReachabilityArea() [determine current location]
                            ├─→ BotGetReachabilityToGoal() [select next edge with routing cost]
                            ├─→ Check timeout & try-count, blacklist if stuck
                            └─→ BotTravel_*() handler (walk, jump, grapple, etc.)
                                 └─→ Dispatch to EA_* actions
                            
Output (fills bot_moveresult_t):
  direction, travel type, velocity override, flags (blocked, moveweight, etc.), weapon/view commands
                            ↓
     Game VM synthesizes usercmd_t from EA_GetInput() accumulator
```

**State carryover**: The movement state (`bot_movestate_t`) persists across frames, allowing handlers to track progress through multi-frame operations (grapple arc, elevator ride, weapon jump timing).

## Learning Notes

### Idiomatic to Early 2000s Engines
- **No ECS**: Per-bot state is monolithic (`bot_movestate_t` struct), not decomposed into components. Modern engines might split grapple state, jump state, etc. into separate systems.
- **Dispatch-based architecture**: Travel handlers are function pointers in a table, not polymorphic objects. Minimal overhead but tightly coupled.
- **Physics by lookup tables**: Jump velocities, gravity, step heights come from libvars set at init time. No per-level tuning or data-driven balancing.
- **Avoid heuristics, not A*** pathfinding: The avoid-reach blacklist is a pragmatic heuristic—it prevents infinite loops but doesn't guarantee alternative routes. A true A*** planner would reconsider the entire path.

### Connections to Modern Concepts
- **Behavior tree / state machine hybrid**: The AI FSM (`ai_dmnet.c`) selects goals; movement FSM (here) executes them. Multiple FSM levels.
- **Blackboard pattern**: Movement state is the blackboard; handlers read/write `reachability_time`, `grapplevisible_time`, etc.
- **Travel type = capability**. The aas_reachability_t's travel type is a capability descriptor—only handlers matching that capability can execute it.

## Potential Issues

1. **Grapple hook stall detection** (GrappleState, ~line 2100): The condition `dist > (lastgrappledist + 16)` (where `lastgrappledist` is the distance to the hook's predicted end point) uses a fixed 16-unit hysteresis. On a very slow or laggy client, the hook may appear stalled but recover after more frames. **Risk**: Could prematurely cancel a valid grapple.

2. **Avoid-reach timeout acceleration** (~line 605): When blocked, the timeout is multiplied by `(AVOIDREACH_TIME - reachabilityTime)`, effectively doubling remaining blacklist time. If the bot is *still blocked after acceleration*, it will retry forever. **Risk**: Infinite loop if a reachability becomes permanently invalid during execution.

3. **Jump prediction model** (`BotWalkInDirection`, `AAS_PredictClientMovement`): Simulates movement assuming perfect input and constant acceleration, not accounting for input rate limiting (cmd.c limits to ~125 Hz), movement stuttering, or network lag. **Risk**: Predicted paths may be unrealistically optimistic.

4. **Mover classification caching** (`modeltypes[]` set once at startup): If a mover's entity type changes during gameplay, the cache becomes stale. **Risk**: Unlikely in Q3A (movers are immutable), but a design fragility.

5. **Fuzzy area retry cost**: Sampling a 3×3×3 grid + AAS_TraceAreas calls can be expensive per-frame when called repeatedly (e.g., standing on edge). No throttling or per-frame budget. **Risk**: Framerate hiccup if many bots are on boundaries.

---

**Summary**: This file exemplifies pragmatic AI engineering circa 2000—heavyweight per-frame dispatch, heuristic avoidance, and optimistic physics prediction. It trades perfect correctness for responsiveness and debuggability.
