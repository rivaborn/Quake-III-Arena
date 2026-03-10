# code/game/g_trigger.c

## File Purpose
Implements all map trigger entities for Quake III Arena's server-side game module. Handles volume-based activation, jump pads, teleporters, hurt zones, and repeating timers that fire targets when players or entities interact with them.

## Core Responsibilities
- Initialize trigger brush entities with correct collision contents and server flags
- Implement `trigger_multiple`: repeatable volume trigger with optional team filtering and wait/random timing
- Implement `trigger_always`: fires targets once on map load, then frees itself
- Implement `trigger_push` / `target_push`: jump pad physics, computing launch velocity to hit a target apex
- Implement `trigger_teleport`: client-predicted teleport volumes, with optional spectator-only mode
- Implement `trigger_hurt`: damage zones with SLOW/SILENT/NO_PROTECTION/START_OFF flags
- Implement `func_timer`: a non-spatial, toggleable repeating timer that fires targets

## Key Types / Data Structures
None defined in this file; relies entirely on `gentity_t`, `gclient_t`, `trace_t`, and `level_locals_t` from `g_local.h`.

## Global / File-Static State
None. All state is stored in per-entity `gentity_t` fields (`nextthink`, `timestamp`, `wait`, `random`, `activator`, etc.).

## Key Functions / Methods

### InitTrigger
- **Signature:** `void InitTrigger(gentity_t *self)`
- **Purpose:** Common setup for all brush-based triggers: sets move direction from angles, loads the brush model, sets contents to `CONTENTS_TRIGGER`, marks entity as `SVF_NOCLIENT`.
- **Inputs:** `self` — the trigger entity
- **Outputs/Return:** void
- **Side effects:** Calls `trap_SetBrushModel`; mutates `self->r.contents`, `self->r.svFlags`, `self->movedir`
- **Calls:** `VectorCompare`, `G_SetMovedir`, `trap_SetBrushModel`

### multi_trigger
- **Signature:** `void multi_trigger(gentity_t *ent, gentity_t *activator)`
- **Purpose:** Core logic for `trigger_multiple`; enforces retrigger cooldown, team spawnflag filtering, fires targets, then schedules next availability or self-destruction.
- **Inputs:** `ent` — the trigger; `activator` — the touching/using entity
- **Outputs/Return:** void
- **Side effects:** Sets `ent->activator`; calls `G_UseTargets`; schedules `multi_wait` or `G_FreeEntity` via `ent->think` / `ent->nextthink`
- **Calls:** `G_UseTargets`, `crandom`
- **Notes:** If `wait < 0`, trigger is one-shot — clears touch callback and self-destructs next frame to avoid freeing during area-link traversal.

### AimAtTarget
- **Signature:** `void AimAtTarget(gentity_t *self)`
- **Purpose:** Calculates the launch velocity (`s.origin2`) for a jump pad so that a projectile launched from the pad's center will reach the apex target entity.
- **Inputs:** `self` — the push trigger; `self->target` names the apex entity
- **Outputs/Return:** void; writes result into `self->s.origin2`
- **Side effects:** Frees `self` via `G_FreeEntity` if target not found or vertical travel time is zero
- **Calls:** `VectorAdd`, `VectorScale`, `G_PickTarget`, `VectorSubtract`, `VectorNormalize`
- **Notes:** Uses kinematic formula `t = sqrt(h / (0.5 * g))`; horizontal speed is `dist / time`; vertical component is `t * gravity`.

### trigger_teleporter_touch
- **Signature:** `void trigger_teleporter_touch(gentity_t *self, gentity_t *other, trace_t *trace)`
- **Purpose:** Touch callback for `trigger_teleport`; validates the activator (alive client, correct spectator flag), looks up destination, and teleports the player.
- **Inputs:** `self` — teleport volume; `other` — touching entity; `trace` — unused
- **Outputs/Return:** void
- **Side effects:** Calls `TeleportPlayer` (moves player, resets physics state)
- **Calls:** `G_PickTarget`, `TeleportPlayer`, `G_Printf`

### hurt_touch
- **Signature:** `void hurt_touch(gentity_t *self, gentity_t *other, trace_t *trace)`
- **Purpose:** Applies periodic damage to any entity inside the hurt trigger volume, respecting SLOW (1 Hz) vs. normal (per-frame) rate, SILENT (no sound), and NO_PROTECTION flags.
- **Inputs:** `self` — hurt volume; `other` — touching entity
- **Side effects:** Updates `self->timestamp`; calls `G_Sound`, `G_Damage`
- **Calls:** `G_Sound`, `G_Damage`

### SP_func_timer
- **Signature:** `void SP_func_timer(gentity_t *self)`
- **Purpose:** Spawns a non-spatial timer entity; if `START_ON`, fires immediately. Clamps random < wait.
- **Side effects:** Sets `self->think`, `self->use`, `self->nextthink`, `self->r.svFlags`

## Control Flow Notes
All `SP_*` functions are called once at map load by the entity spawning system (`G_SpawnEntitiesFromString`). Touch callbacks are invoked per-frame from `G_TouchTriggers` during `ClientThink`. Think callbacks (`multi_wait`, `trigger_always_think`, `AimAtTarget`, `func_timer_think`) run during `G_RunThink` each server frame.

## External Dependencies
- **`g_local.h`**: `gentity_t`, `gclient_t`, `level_locals_t` (`level`), `g_gravity`, `FRAMETIME`, `CONTENTS_TRIGGER`, `SVF_NOCLIENT`, `TEAM_RED/BLUE/SPECTATOR`, `ET_PUSH_TRIGGER`, `ET_TELEPORT_TRIGGER`, damage flags, `MOD_TRIGGER_HURT`
- **Defined elsewhere:** `G_UseTargets`, `G_PickTarget`, `G_FreeEntity`, `G_SetMovedir`, `G_Sound`, `G_SoundIndex`, `G_Damage`, `TeleportPlayer`, `BG_TouchJumpPad`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_SetBrushModel`, `crandom`, `G_SpawnFloat`, `G_Printf`
