# code/game/g_mover.c

## File Purpose
Implements all moving entity (mover) logic for Quake III Arena's game module, including the push/collision system for movers and spawn functions for doors, platforms, buttons, trains, and decorative movers (rotating, bobbing, pendulum, static).

## Core Responsibilities
- Execute per-frame movement for mover entities via `G_RunMover` / `G_MoverTeam`
- Push (or block) entities that intersect a moving brush, with full rollback on failure
- Manage binary mover state transitions (POS1 ↔ POS2) and associated sounds/events
- Spawn and configure all `func_*` mover entity types from map data
- Handle door trigger volumes, spectator teleportation through doors, and platform touch logic
- Synchronize team-linked mover slaves so all parts move atomically

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `pushed_t` | struct | Saves an entity's pre-push position/angles/deltayaw for rollback |
| `moverState_t` | enum (from g_local.h) | Four-state machine: `MOVER_POS1`, `MOVER_POS2`, `MOVER_1TO2`, `MOVER_2TO1` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pushed[MAX_GENTITIES]` | `pushed_t[]` | global | Stack of saved entity positions for push rollback |
| `pushed_p` | `pushed_t *` | global | Stack pointer into `pushed[]`; reset at the start of each `G_MoverTeam` call |

## Key Functions / Methods

### G_TestEntityPosition
- **Signature:** `gentity_t *G_TestEntityPosition( gentity_t *ent )`
- **Purpose:** Tests whether an entity is currently embedded in solid geometry.
- **Inputs:** Entity to test.
- **Outputs/Return:** Blocking entity if start-solid, `NULL` if clear.
- **Side effects:** Issues a `trap_Trace` call.
- **Calls:** `trap_Trace`
- **Notes:** Uses `ent->client->ps.origin` for players, `s.pos.trBase` for non-clients.

### G_TryPushingEntity
- **Signature:** `qboolean G_TryPushingEntity( gentity_t *check, gentity_t *pusher, vec3_t move, vec3_t amove )`
- **Purpose:** Attempts to move one entity by a mover's linear + angular displacement; saves state for rollback.
- **Inputs:** Entity to push, the pushing brush, translation delta, rotation delta.
- **Outputs/Return:** `qtrue` if push succeeded; `qfalse` if blocked.
- **Side effects:** Modifies `check->s.pos.trBase`, `check->client->ps.origin`, `groundEntityNum`, `delta_angles[YAW]`; appends to `pushed[]`; calls `trap_LinkEntity`.
- **Calls:** `G_CreateRotationMatrix`, `G_TransposeMatrix`, `G_RotatePoint`, `G_TestEntityPosition`, `trap_LinkEntity`
- **Notes:** `EF_MOVER_STOP` entities only block, not push, unless the check entity is riding on top.

### G_MoverPush
- **Signature:** `qboolean G_MoverPush( gentity_t *pusher, vec3_t move, vec3_t amove, gentity_t **obstacle )`
- **Purpose:** Core push dispatcher — finds all entities in the swept volume and attempts to push each one; rolls back all on failure.
- **Inputs:** Pusher brush, linear move, angular move, out-pointer for blocking entity.
- **Outputs/Return:** `qtrue` if all pushes succeeded.
- **Side effects:** Temporarily unlinks/re-links pusher; modifies `pusher->r.currentOrigin/currentAngles`; can call `G_Damage`, `G_ExplodeMissile`, `G_FreeEntity`.
- **Calls:** `trap_UnlinkEntity`, `trap_EntitiesInBox`, `trap_LinkEntity`, `G_TryPushingEntity`, `G_TryPushingProxMine`, `G_CheckProxMinePosition`, `G_Damage`, `G_ExplodeMissile`, `G_AddEvent`, `G_FreeEntity`
- **Notes:** TR_SINE bobbing movers are instant-kill and never blocked. `#ifdef MISSIONPACK` block handles proximity mine interaction.

### G_MoverTeam
- **Signature:** `void G_MoverTeam( gentity_t *ent )`
- **Purpose:** Atomically moves all entities in a mover team; rolls back if any part is blocked, then calls the blocked callback or reached callback.
- **Inputs:** Team captain entity.
- **Side effects:** Resets `pushed_p`; updates `r.currentOrigin/currentAngles` for all slaves; calls `ent->blocked` or `part->reached`; calls `trap_LinkEntity`.
- **Calls:** `BG_EvaluateTrajectory`, `G_MoverPush`, `trap_LinkEntity`

### G_RunMover
- **Signature:** `void G_RunMover( gentity_t *ent )`
- **Purpose:** Per-frame entry point for mover entities; skips slaves and stationary movers.
- **Calls:** `G_MoverTeam`, `G_RunThink`

### SetMoverState
- **Signature:** `void SetMoverState( gentity_t *ent, moverState_t moverState, int time )`
- **Purpose:** Transitions a mover to a new state, configuring trajectory type/base/delta accordingly.
- **Calls:** `BG_EvaluateTrajectory`, `trap_LinkEntity`

### InitMover
- **Signature:** `void InitMover( gentity_t *ent )`
- **Purpose:** Common initializer for all binary movers — sets model2, loop sound, constant light, `use`/`reached` callbacks, trajectory, and travel duration from speed.
- **Calls:** `G_ModelIndex`, `G_SoundIndex`, `G_SpawnString`, `G_SpawnFloat`, `G_SpawnVector`, `trap_LinkEntity`

### Use_BinaryMover
- **Signature:** `void Use_BinaryMover( gentity_t *ent, gentity_t *other, gentity_t *activator )`
- **Purpose:** Handles activation events on binary movers, supporting mid-travel reversal in all four states.
- **Calls:** `MatchTeam`, `G_AddEvent`, `trap_AdjustAreaPortalState`

### SP_func_door / SP_func_plat / SP_func_button / SP_func_train / SP_func_rotating / SP_func_bobbing / SP_func_pendulum / SP_func_static
- Spawn functions called once at map load for each respective entity type. Each sets type-specific parameters, calls `trap_SetBrushModel`, and calls `InitMover`. Door and plat also spawn trigger volumes.

### Notes (minor helpers)
- `G_CreateRotationMatrix`, `G_TransposeMatrix`, `G_RotatePoint` — matrix math helpers for rotating-mover entity displacement.
- `ReturnToPos1`, `Reached_BinaryMover`, `MatchTeam` — callbacks and team sync helpers for binary mover lifecycle.
- `Think_SpawnNewDoorTrigger`, `Think_MatchTeam`, `Think_SetupTrainTargets`, `Think_BeginMoving` — deferred think callbacks.
- `Reached_Train`, `Touch_Plat`, `Touch_DoorTrigger`, `Touch_DoorTriggerSpectator`, `Blocked_Door` — event callbacks for specific mover types.

## Control Flow Notes
- `G_RunMover` is called each server frame from the entity think dispatch (`G_RunThink` chain).
- Only the team master runs `G_MoverTeam`; slaves are dragged along atomically.
- Spawn functions (`SP_func_*`) execute once during `G_SpawnEntitiesFromString` at map load; they set `think`/`nextthink` to defer trigger and train setup by one frame (`FRAMETIME = 100 ms`).
- Mover trajectory is evaluated by the shared `BG_EvaluateTrajectory` (client/server common), keeping client prediction consistent.

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `g_entities[]`, `level` (level_locals_t), `BG_EvaluateTrajectory`, `RadiusFromBounds`, `AngleVectors`, `VectorInverse`, `trap_*` syscalls, `G_Damage`, `G_AddEvent`, `G_UseTargets`, `G_Find`, `G_Spawn`, `G_FreeEntity`, `TeleportPlayer`, `Team_DroppedFlagThink`, `G_ExplodeMissile`, `G_RunThink`, `g_gravity` (vmCvar_t)
