# code/game/be_ai_move.h — Enhanced Analysis

## Architectural Role

This header defines the **public movement AI interface** exposed by botlib to the game VM. It acts as the boundary contract between the engine's bot lifecycle management (in `code/server/sv_bot.c`) and the self-contained navigation/pathfinding layer (`code/botlib/be_ai_move.c`). The game VM calls these functions through the `trap_BotLib*` syscall range (opcodes 200–599), making this file part of botlib's external API surface—not an internal implementation detail. Every per-frame bot think cycle funnels through the state-synchronization → movement computation → result-extraction pattern defined here.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/server/sv_bot.c`**: Main bot driver; calls `BotAllocMoveState`, `BotInitMoveState`, `BotMoveToGoal` each frame via `trap_BotLib*` syscalls; interprets `bot_moveresult_t` flags to build `usercmd_t` input commands
- **`code/game/g_bot.c`**: Server-side bot spawn/connect/disconnect lifecycle; allocates move state on bot spawn, frees on disconnect
- **`code/game/ai_dmnet.c`**: High-level bot FSM that queries movement capabilities via `BotMovementViewTarget`, `BotPredictVisiblePosition`, and navigation goals into `BotMoveToGoal`
- **`code/botlib/be_ai_main.c`**: Implements the botlib entry point; exports movement setup/shutdown hooks
- **`code/botlib/be_interface.c`**: Syscall dispatcher; marshals `BotAllocMoveState` / `BotFreeMoveState` / `BotInitMoveState` / `BotMoveToGoal` calls into botlib function pointers

### Outgoing (what this file depends on)
- **`code/game/be_ai_goal.h`**: Declares `bot_goal_t`, the navigation target struct passed to `BotMoveToGoal`
- **`code/game/q_shared.h`**: Provides `vec3_t` vector type and primitive types
- **`code/botlib/be_ai_move.c`**: Implementation file; houses internal move state pool, frame-by-frame physics, obstacle/avoid-spot logic, travel-type execution
- **Engine/player state boundary**: `bot_initmove_t.or_moveflags` **receives** `MFL_ONGROUND`, `MFL_TELEPORTED`, `MFL_WATERJUMP` from the engine's `playerState_t` each frame, binding botlib state to the authoritative player state

## Design Patterns & Rationale

1. **Handle-Based Opaque State Management**: Move states are allocated by integer handle, not pointer-based. This isolates botlib's internal pool layout from the game VM, preventing accidental corruption and enabling future reimplementation (e.g., hash table vs. array) without API change.

2. **Per-Frame Explicit Synchronization**: `BotInitMoveState` is called **every frame** with fresh `bot_initmove_t` data from the current `playerState_t`. This avoids stale internal state and ensures the bot's cached location/velocity always matches the engine's ground truth. Compare: many engines use "push state once, query many times"; here Q3 uses "sync every frame, then compute once."

3. **Result Flags as Output Metadata**: `bot_moveresult_t.flags` uses a bitmask vocabulary (`MOVERESULT_MOVEMENTVIEW`, `MOVERESULT_BLOCKED`, etc.) to communicate **what the bot is doing** rather than encoding detailed state. The game layer uses these flags to decide: "Should I use the ideal view angles?" (check `MOVERESULT_MOVEMENTVIEW`), "Is the bot stuck?" (check `MOVERESULT_BLOCKEDBYAVOIDSPOT`), etc. This decouples game-level decision logic from botlib internals.

4. **Closed Vocabulary for Movement Types**: The `MOVE_*` bitmask set is fixed (walk, crouch, jump, grapple, rocket jump, BFG jump). Rather than allowing arbitrary movement types, botlib defines the enumerated capabilities it can execute. Game code must work within this constraint.

5. **Spatial Hazard Avoidance as Separation of Concerns**: `BotAddAvoidSpot` is a minimal per-bot callback to mark regions (origin + radius + type) that the pathfinder should route around. This avoids hardcoding hazard knowledge into the core movement FSM; external code (e.g., game logic detecting a mine field) can dynamically request avoidance without modifying botlib.

## Data Flow Through This File

```
Per-Frame Input:
  playerState_t (origin, velocity, viewoffset, entity/client num, view angles)
    ↓ (copied into bot_initmove_t by game/server code)
  BotInitMoveState( handle, initmove )
    ↓ (internal: sync bot's position, velocity, view state)
  [Internal bot state pool updated]

Movement Query:
  bot_goal_t (target area, position, flags)
    ↓
  BotMoveToGoal( result, handle, goal, travelflags )
    ↓ (internal: pathfind via AAS graph, execute travel type, update position)
  [Internal: may call AAS_* for reachability, perform physics simulation]
    ↓
  bot_moveresult_t populated
    ↓ (direction, ideal angles, blocked status, weapon override, flags)

Output → Game Layer:
  Game code reads result:
    - result->movedir  → feed into usercmd_t forward/strafe/up
    - result->ideal_viewangles + MOVERESULT_MOVEMENTVIEW → override view
    - result->weapon + MOVERESULT_MOVEMENTWEAPON → fire weapon for mobility
    - result->blocked + result->type → detect stuck/wait states
```

**Key observation**: The `bot_moveresult_t` is the only output channel. No internal state is exposed; the game layer receives only what it needs to synthesize `usercmd_t` and detect anomalies (blocked, waiting, etc.).

## Learning Notes

- **Idiomatic to Early-2000s Game AI**: This design reflects pre-modern AI trends. Modern engines (Unreal, Unity) use behavior trees or planners that can operate on the render thread; here, botlib is a **deterministic, frame-locked subsystem** that must be called synchronously each server frame, with no multi-threading. The handle-based API avoids mutexes.
  
- **AAS as the Navigation Substrate**: Unlike modern engines that use navmeshes or grid-based pathfinding, Q3 uses **Annotated Areal Space (AAS)**—a geometric decomposition pre-computed offline by `bspc`. Every `BotMoveToGoal` call implicitly trusts the AAS graph; if reachability data is stale or invalid, bots fail silently. This is a design tradeoff: AAS is memory-efficient and permits complex travel types (jump pads, teleporters, grapple points) but is offline and map-specific.

- **Per-Bot vs. Global State**: Move states are per-bot (one per client), but avoid-spots are per-movestate and capped at `MAX_AVOIDSPOTS=32`. This scales linearly; dynamic hazard markers must be managed by the caller (game code), not botlib.

- **Movement as Deterministic Physics**: `BotInitMoveState` + `BotMoveToGoal` form a **closed-loop simulation**. The bot's position, velocity, and view angles are all computed internally; the game layer never directly modifies them. This ensures replay consistency and prevents desync bugs.

## Potential Issues

1. **No Async or Predictive Loading**: If a new map is loaded mid-session, `BotSetBrushModelTypes` must be called to refresh brush collision types. Forgetting this call leaves bots unable to traverse dynamic obstacles. Modern engines would integrate map-load into the API contract.

2. **Avoid-Spot Overflow Silently**: If `BotAddAvoidSpot` is called >32 times per bot per frame without `AVOID_CLEAR`, the oldest spots are dropped silently (no error). Game code must manage this quota or risk silent degradation.

3. **Result Flags as Implicit State Machine**: The caller (game code) must understand the semantics of each flag combination. For example, `MOVERESULT_BLOCKED | MOVERESULT_BLOCKEDBYAVOIDSPOT` means "stuck by avoid-spot," not just "blocked." There's no explicit documentation of valid flag combinations in this header.

4. **View Angle Validity Only When Flag Set**: The comment `//NOTE: the ideal_viewangles are only valid if MFL_MOVEMENTVIEW is set` is easy to miss. Game code that always trusts `result->ideal_viewangles` will misalign the bot's view during non-movement phases (combat, waiting).
