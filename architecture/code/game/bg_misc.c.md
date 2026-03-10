# code/game/bg_misc.c

## File Purpose
Defines the master item registry (`bg_itemlist`) for all pickups in Quake III Arena and provides stateless utility functions shared between the server game and client game modules for item lookup, trajectory evaluation, player state conversion, and event management.

## Core Responsibilities
- Declares and initializes the global `bg_itemlist[]` array containing every item definition (weapons, ammo, armor, health, powerups, holdables, team items)
- Provides item lookup functions by powerup tag, holdable tag, weapon tag, and pickup name
- Implements trajectory position and velocity evaluation for all `trType_t` variants
- Determines whether a player can pick up a given item (`BG_CanItemBeGrabbed`) with full gametype/team/MISSIONPACK awareness
- Tests spatial proximity between a player and an item entity
- Manages the predictable event ring-buffer in `playerState_t`
- Converts `playerState_t` → `entityState_t` (both interpolated and extrapolated variants)
- Handles jump-pad velocity application and event generation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `gitem_t` | struct (typedef) | Per-item descriptor: classname, sounds, models, icon, pickup name, quantity, type, tag, precache/sound strings |
| `itemType_t` | enum | Item category flags (`IT_WEAPON`, `IT_AMMO`, `IT_ARMOR`, `IT_HEALTH`, `IT_POWERUP`, `IT_HOLDABLE`, `IT_PERSISTANT_POWERUP`, `IT_TEAM`) |
| `trajectory_t` | struct | Parametric motion descriptor used by both entities and the trajectory evaluators |
| `playerState_t` | struct | Full authoritative player state; source for entity state conversion |
| `entityState_t` | struct | Network-transmitted entity snapshot; target of player state conversion |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `bg_itemlist[]` | `gitem_t[]` | global | Master registry of all game items; index 0 is a null sentinel |
| `bg_numItems` | `int` | global | Count of valid entries (`sizeof(bg_itemlist)/sizeof([0]) - 1`) |
| `eventnames[]` | `char *[]` | file-static (translation unit) | String table mapping `entity_event_t` values to names for debug logging |

## Key Functions / Methods

### BG_FindItemForPowerup
- Signature: `gitem_t *BG_FindItemForPowerup( powerup_t pw )`
- Purpose: Linear scan of `bg_itemlist` for an item whose type is `IT_POWERUP`, `IT_TEAM`, or `IT_PERSISTANT_POWERUP` and whose `giTag` matches `pw`.
- Inputs: `pw` — powerup enum value
- Outputs/Return: Pointer into `bg_itemlist`, or `NULL` if not found
- Side effects: None
- Calls: None
- Notes: Returns `NULL` on miss (no fatal error); callers must handle.

### BG_FindItemForHoldable
- Signature: `gitem_t *BG_FindItemForHoldable( holdable_t pw )`
- Purpose: Finds the `IT_HOLDABLE` item matching tag `pw`.
- Inputs: `pw` — holdable enum value
- Outputs/Return: Pointer into `bg_itemlist`
- Side effects: Calls `Com_Error(ERR_DROP, ...)` if not found
- Calls: `Com_Error`
- Notes: Fatal on miss — a missing holdable is treated as a programming error.

### BG_FindItemForWeapon
- Signature: `gitem_t *BG_FindItemForWeapon( weapon_t weapon )`
- Purpose: Finds the `IT_WEAPON` item matching `weapon`.
- Inputs: `weapon` — weapon enum value
- Outputs/Return: Pointer into `bg_itemlist`
- Side effects: `Com_Error(ERR_DROP)` if not found
- Calls: `Com_Error`

### BG_FindItem
- Signature: `gitem_t *BG_FindItem( const char *pickupName )`
- Purpose: Case-insensitive search by `pickup_name` string.
- Inputs: `pickupName` — display/pickup name
- Outputs/Return: Pointer into `bg_itemlist` or `NULL`
- Side effects: None
- Calls: `Q_stricmp`

### BG_PlayerTouchesItem
- Signature: `qboolean BG_PlayerTouchesItem( playerState_t *ps, entityState_t *item, int atTime )`
- Purpose: AABB proximity test (asymmetric box) to determine if a player overlaps an item for pickup; used by both client prediction and server.
- Inputs: `ps` — player state with origin; `item` — item entity state; `atTime` — time to evaluate item position
- Outputs/Return: `qtrue` if within pickup range
- Side effects: None
- Calls: `BG_EvaluateTrajectory`
- Notes: Ducked-height difference is intentionally ignored.

### BG_CanItemBeGrabbed
- Signature: `qboolean BG_CanItemBeGrabbed( int gametype, const entityState_t *ent, const playerState_t *ps )`
- Purpose: Authoritative pickup eligibility check; must be identical on server and client to avoid prediction errors. Handles ammo caps, armor caps (with MISSIONPACK handicap), health limits, CTF/1FCTF flag rules, team-only persistant powerups, and holdable slot limits.
- Inputs: `gametype` — current game mode; `ent` — item entity (index into `bg_itemlist` via `modelindex`); `ps` — player's current state
- Outputs/Return: `qtrue` if the item can be picked up
- Side effects: `Com_Error(ERR_DROP)` on out-of-range index or `IT_BAD`
- Calls: `Com_Error`, `Com_Printf`
- Notes: Ammo cap hard-coded at 200. MISSIONPACK conditionals alter armor/health upper bounds based on persistent powerup.

### BG_EvaluateTrajectory
- Signature: `void BG_EvaluateTrajectory( const trajectory_t *tr, int atTime, vec3_t result )`
- Purpose: Computes world-space position at `atTime` for a given trajectory type.
- Inputs: `tr` — trajectory descriptor; `atTime` — game time in milliseconds
- Outputs/Return: `result` — computed position
- Side effects: `Com_Error` on unknown `trType`
- Calls: `VectorCopy`, `VectorMA`, `sin`, `Com_Error`
- Notes: `TR_GRAVITY` uses `DEFAULT_GRAVITY` (800); comment marks local gravity as unimplemented.

### BG_EvaluateTrajectoryDelta
- Signature: `void BG_EvaluateTrajectoryDelta( const trajectory_t *tr, int atTime, vec3_t result )`
- Purpose: Computes instantaneous velocity (derivative of position) for a trajectory at `atTime`.
- Inputs/Outputs: same pattern as `BG_EvaluateTrajectory`
- Calls: `VectorClear`, `VectorCopy`, `VectorScale`, `cos`, `Com_Error`
- Notes: `TR_SINE` derivative scales by 0.5 (amplitude factor).

### BG_AddPredictableEventToPlayerstate
- Signature: `void BG_AddPredictableEventToPlayerstate( int newEvent, int eventParm, playerState_t *ps )`
- Purpose: Appends an event into the two-slot ring buffer in `playerState_t`, incrementing `eventSequence`.
- Side effects: Modifies `ps->events[]`, `ps->eventParms[]`, `ps->eventSequence`
- Calls: `trap_Cvar_VariableStringBuffer` (debug only), `Com_Printf` (debug only)

### BG_TouchJumpPad
- Signature: `void BG_TouchJumpPad( playerState_t *ps, entityState_t *jumppad )`
- Purpose: Applies jump-pad velocity to player and generates `EV_JUMP_PAD` event; skips spectators and flying players; deduplicates within a frame.
- Side effects: Modifies `ps->velocity`, `ps->jumppad_ent`, `ps->jumppad_frame`; queues predictable event
- Calls: `vectoangles`, `fabs`, `AngleNormalize180`, `BG_AddPredictableEventToPlayerstate`, `VectorCopy`

### BG_PlayerStateToEntityState
- Signature: `void BG_PlayerStateToEntityState( playerState_t *ps, entityState_t *s, qboolean snap )`
- Purpose: Projects authoritative player state into a network-transmittable entity state using `TR_INTERPOLATE`.
- Side effects: Overwrites `*s` entirely
- Calls: `VectorCopy`, `SnapVector`

### BG_PlayerStateToEntityStateExtraPolate
- Signature: `void BG_PlayerStateToEntityStateExtraPolate( playerState_t *ps, entityState_t *s, int time, qboolean snap )`
- Purpose: Same as above but uses `TR_LINEAR_STOP` with a 50 ms extrapolation window for smoother remote player movement.
- Notes: `trDuration` of 50 ms corresponds to `1000/sv_fps` at default 20 Hz server tick.

## Control Flow Notes
This file has **no frame loop participation of its own**. It is a pure utility module:
- Called during **pmove** (server per-tick and client prediction) for trajectory evaluation and jump-pad handling.
- Called during **snapshot generation** (`BG_PlayerStateToEntityState*`) after each usercmd execution.
- Called at **item spawn/pickup time** (both server and cgame) for item lookup and grab eligibility.
- `bg_itemlist` is read at **map init** time for precaching and entity spawning.

## External Dependencies
- `q_shared.h` — core math macros (`VectorCopy`, `VectorMA`, `VectorScale`, `VectorClear`, `SnapVector`), type definitions (`vec3_t`, `playerState_t`, `entityState_t`, `trajectory_t`, `qboolean`), `Com_Error`, `Com_Printf`, `Q_stricmp`, `AngleNormalize180`, `vectoangles`
- `bg_public.h` — `gitem_t`, `itemType_t`, `powerup_t`, `holdable_t`, `weapon_t`, `entity_event_t`, `gametype_t`, `DEFAULT_GRAVITY`, `GIB_HEALTH`, `STAT_*`, `PERS_*`, `PW_*`, `HI_*`, `WP_*`, `EV_*`, `ET_*`, `TR_*`
- `trap_Cvar_VariableStringBuffer` — declared (not defined) in this file; resolved at link time against the VM trap table (cgame or game module)
- `sin`, `cos`, `fabs` — C math library (or VM substitutes via `bg_lib`)
