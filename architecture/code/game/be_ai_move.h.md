# code/game/be_ai_move.h

## File Purpose
Public header defining the movement AI interface for Quake III's bot library. It declares movement type flags, move state flags, result flags, key data structures, and the full function API used by game code to drive bot locomotion.

## Core Responsibilities
- Define bitmask constants for movement types (walk, crouch, jump, grapple, rocket jump)
- Define bitmask constants for movement state flags (on-ground, swimming, teleported, etc.)
- Define bitmask constants for movement result flags (view override, blocked, obstacle, elevator)
- Declare `bot_initmove_t` for seeding a move state from player/entity state
- Declare `bot_moveresult_t` for communicating locomotion outcomes back to callers
- Declare `bot_avoidspot_t` for spatial hazard avoidance regions
- Expose the full movement AI lifecycle API (alloc/init/move/free)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_initmove_t` | struct | Seeds a move state each think tick: origin, velocity, view offset, entity/client numbers, think time, presence type, view angles, and OR'd move flags from `playerState_t` |
| `bot_moveresult_t` | struct | Output of a movement tick: failure status, block info, last travel type, result flags, weapon override, movement direction, and ideal view angles |
| `bot_avoidspot_t` | struct | Spherical region the bot should avoid, parameterized by origin, radius, and avoidance type |

## Global / File-Static State
None.

## Key Functions / Methods

### BotAllocMoveState / BotFreeMoveState
- Signature: `int BotAllocMoveState(void)` / `void BotFreeMoveState(int handle)`
- Purpose: Lifecycle management — allocate and release a per-bot movement state slot.
- Inputs: None / integer handle returned by alloc.
- Outputs/Return: Alloc returns an integer handle; free returns void.
- Side effects: Modifies internal move state pool (defined in `be_ai_move.c`).
- Calls: Not inferable from this file.
- Notes: One state per bot client; handle is opaque to callers.

### BotInitMoveState
- Signature: `void BotInitMoveState(int handle, bot_initmove_t *initmove)`
- Purpose: Synchronizes a bot's movement state with current entity/player state at the start of each think frame.
- Inputs: Handle to move state; pointer to `bot_initmove_t` populated from `playerState_t`.
- Outputs/Return: void.
- Side effects: Mutates internal move state referenced by handle.
- Calls: Not inferable from this file.
- Notes: `or_moveflags` fields `MFL_ONGROUND`, `MFL_TELEPORTED`, `MFL_WATERJUMP` must come from the engine's player state.

### BotMoveToGoal
- Signature: `void BotMoveToGoal(bot_moveresult_t *result, int movestate, bot_goal_t *goal, int travelflags)`
- Purpose: Core per-frame locomotion driver; steers the bot toward a navigation goal via the AAS reachability graph.
- Inputs: Output result struct pointer; move state handle; goal descriptor; bitmask of allowed travel types.
- Outputs/Return: Populates `*result` with direction, flags, weapon, and view angles.
- Side effects: May trigger bot entity actions (jump, crouch, fire grapple); updates internal avoid-reach tracking.
- Calls: Not inferable from this file.
- Notes: `result->ideal_viewangles` is only valid when `MOVERESULT_MOVEMENTVIEW` is set in `result->flags`.

### BotMoveInDirection
- Signature: `int BotMoveInDirection(int movestate, vec3_t dir, float speed, int type)`
- Purpose: Commands a bot to move in an explicit world-space direction at a given speed using a specified movement type.
- Inputs: Move state handle; direction vector; speed scalar; movement type bitmask (`MOVE_WALK`, `MOVE_JUMP`, etc.).
- Outputs/Return: Non-zero on success.
- Side effects: Schedules bot action inputs.
- Notes: Bypass of goal-based navigation; used for obstacle avoidance or scripted maneuvers.

### BotReachabilityArea / BotMovementViewTarget / BotPredictVisiblePosition
- **`BotReachabilityArea`**: Returns the AAS area number the given origin falls within for the specified client.
- **`BotMovementViewTarget`**: Computes a look-at point along the path toward a goal, used to orient the bot's aim during movement.
- **`BotPredictVisiblePosition`**: Predicts a future world position where a target moving toward a goal will be visible from a given origin/area.

### BotAddAvoidSpot
- Signature: `void BotAddAvoidSpot(int movestate, vec3_t origin, float radius, int type)`
- Purpose: Registers a spatial hazard sphere the bot should route around or never fully block at. Passing `AVOID_CLEAR` removes all spots.
- Side effects: Mutates avoid-spot list in the named move state (max `MAX_AVOIDSPOTS` = 32).

### BotSetupMoveAI / BotShutdownMoveAI / BotSetBrushModelTypes
- Lifecycle hooks: setup/shutdown the global movement AI subsystem and refresh brush-model type classification on map load.

## Control Flow Notes
Called from the game-side bot think loop: `BotInitMoveState` is called first each frame to sync player state, then `BotMoveToGoal` (or `BotMoveInDirection`) produces a `bot_moveresult_t` which the game layer translates into `usercmd_t` inputs. `BotSetupMoveAI` / `BotShutdownMoveAI` bracket the server session; `BotSetBrushModelTypes` must be called on every map change.

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `bot_goal_t` — defined in `be_ai_goal.h`
- `bot_initmove_t.or_moveflags` values (`MFL_ONGROUND`, etc.) sourced from engine `playerState_t` by the caller
- Implementation: `code/botlib/be_ai_move.c`
