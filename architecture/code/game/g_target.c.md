# code/game/g_target.c

## File Purpose
Implements all `target_*` entity types for Quake III Arena's server-side game logic. These are invisible map entities that perform actions (give items, print messages, play sounds, fire lasers, teleport players, etc.) when triggered by other entities or players.

## Core Responsibilities
- Register spawn functions (`SP_target_*`) for each target entity class
- Assign `use` callbacks that execute when the entity is triggered
- Implement delayed firing, score modification, and message broadcasting
- Manage looping/one-shot audio via `target_speaker`
- Operate a continuous damage laser (`target_laser`) with per-frame think logic
- Teleport activating players to a named destination entity
- Link `target_location` entities into a global linked list for HUD location display

## Key Types / Data Structures
None defined in this file; relies entirely on `gentity_t`, `gclient_t`, and `level_locals_t` from `g_local.h`.

## Global / File-Static State
None. All state is stored on individual `gentity_t` fields or the global `level` struct.

## Key Functions / Methods

### Use_Target_Give
- **Signature:** `void Use_Target_Give(gentity_t *ent, gentity_t *other, gentity_t *activator)`
- **Purpose:** Iterates all entities named by `ent->target`, calls `Touch_Item` on each to give the item to the activator, then suppresses respawn.
- **Inputs:** `ent` — the give entity; `activator` — must have a valid `client`.
- **Outputs/Return:** void
- **Side effects:** Modifies activator inventory via `Touch_Item`; sets `t->nextthink = 0`; unlinks item entities.
- **Calls:** `G_Find`, `Touch_Item`, `trap_UnlinkEntity`
- **Notes:** Silently no-ops if activator has no client or `ent->target` is NULL.

### Use_target_remove_powerups
- **Signature:** `void Use_target_remove_powerups(gentity_t *ent, gentity_t *other, gentity_t *activator)`
- **Purpose:** Strips all powerups from the activator; returns CTF flags to their team first.
- **Side effects:** Calls `Team_ReturnFlag` if a flag powerup is held; zeroes `ps.powerups`.
- **Calls:** `Team_ReturnFlag`

### Use_Target_Delay / Think_Target_Delay
- **Signature:** `void Use_Target_Delay(gentity_t *ent, gentity_t *other, gentity_t *activator)`
- **Purpose:** Schedules a deferred `G_UseTargets` call after `wait ± random` seconds.
- **Side effects:** Sets `ent->nextthink`, `ent->think`, `ent->activator`.
- **Calls:** `G_UseTargets` (via think), `crandom`
- **Notes:** `SP_target_delay` enforces a minimum wait of 1 second; supports legacy `"delay"` key for backwards compatibility.

### Use_Target_Speaker
- **Signature:** `void Use_Target_Speaker(gentity_t *ent, gentity_t *other, gentity_t *activator)`
- **Purpose:** Toggles looping sounds or fires one-shot general/global sound events.
- **Side effects:** Modifies `ent->s.loopSound`; calls `G_AddEvent` on entity or activator.
- **Calls:** `G_AddEvent`
- **Notes:** Spawnflag 1/2 = looped; 4 = global; 8 = activator-relative.

### SP_target_speaker
- **Signature:** `void SP_target_speaker(gentity_t *ent)`
- **Purpose:** Full initialization: resolves sound path, registers sound index, configures entity state for client-side repeating playback, links entity.
- **Side effects:** Sets `ent->s.eType = ET_SPEAKER`; calls `trap_LinkEntity`; may set `SVF_BROADCAST`.
- **Calls:** `G_SpawnFloat`, `G_SpawnString`, `G_SoundIndex`, `trap_LinkEntity`, `G_Error`

### target_laser_think
- **Signature:** `void target_laser_think(gentity_t *self)`
- **Purpose:** Per-frame think: traces a ray from laser origin along `movedir`, damages the first entity hit, updates visual endpoint.
- **Side effects:** Calls `G_Damage` on hit entity; updates `self->s.origin2`; re-links entity; schedules next think in `FRAMETIME`.
- **Calls:** `VectorMA`, `VectorSubtract`, `VectorNormalize`, `trap_Trace`, `G_Damage`, `trap_LinkEntity`

### target_laser_start
- **Signature:** `void target_laser_start(gentity_t *self)`
- **Purpose:** Post-spawn initialization: resolves target entity or angle direction, assigns use/think callbacks, applies START_ON flag.
- **Calls:** `G_Find`, `G_SetMovedir`, `target_laser_on` / `target_laser_off`
- **Notes:** Deferred one frame from `SP_target_laser` to allow all entities to spawn first.

### target_teleporter_use
- **Signature:** `void target_teleporter_use(gentity_t *self, gentity_t *other, gentity_t *activator)`
- **Purpose:** Picks a random entity matching `self->target` as destination, teleports the activating player there.
- **Calls:** `G_PickTarget`, `TeleportPlayer`

### target_relay_use
- **Signature:** `void target_relay_use(gentity_t *self, gentity_t *other, gentity_t *activator)`
- **Purpose:** Conditionally forwards trigger to sub-targets, optionally filtering by team or firing only one random target.
- **Calls:** `G_PickTarget`, `G_UseTargets`

### target_location_linkup
- **Signature:** `static void target_location_linkup(gentity_t *ent)`
- **Purpose:** One-time initialization that chains all `target_location` entities into `level.locationHead` and registers their names as config strings.
- **Side effects:** Sets `level.locationLinked = qtrue`; calls `trap_SetConfigstring` for `CS_LOCATIONS + n`; overwrites `ent->health` as a location index.
- **Notes:** Uses `ent->nextTrain` as link pointer (repurposing mover field). Guard against double-init via `level.locationLinked`.

## Control Flow Notes
All `SP_target_*` functions run during map entity spawning. `use` callbacks fire at arbitrary game time when another entity calls `G_UseTargets` or directly invokes `ent->use`. `target_laser` enters the per-frame think loop immediately after spawn (or on activation). `target_location_linkup` runs 200 ms after spawn to ensure all locations are spawned before linking.

## External Dependencies
- **`g_local.h`** — `gentity_t`, `level_locals_t`, `gclient_t`, all trap/utility declarations
- **Defined elsewhere:** `Touch_Item` (g_items.c), `Team_ReturnFlag` (g_team.c), `G_UseTargets`, `G_Find`, `G_PickTarget`, `G_SetMovedir`, `G_AddEvent`, `G_SoundIndex`, `G_SetOrigin` (g_utils.c), `TeleportPlayer` (g_misc.c), `G_Damage` (g_combat.c), `AddScore` (g_client.c), `G_TeamCommand` (g_utils.c), all `trap_*` syscalls
