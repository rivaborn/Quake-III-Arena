# code/game/g_misc.c

## File Purpose
Implements miscellaneous map entity spawn functions and gameplay systems for the Quake III Arena game module, including teleportation logic, portal surfaces, positional markers, and trigger-based weapon shooters.

## Core Responsibilities
- Spawn and initialize editor-only or utility entities (`info_null`, `info_camp`, `light`, `func_group`)
- Implement the `TeleportPlayer` function used by trigger teleporters and portals
- Set up portal surface/camera pairs for in-world mirror/portal rendering
- Initialize trigger-based weapon shooter entities (`shooter_rocket`, `shooter_plasma`, `shooter_grenade`)
- Handle `#ifdef MISSIONPACK` portal item mechanics (drop source/destination pads)

## Key Types / Data Structures
None defined in this file; relies entirely on `gentity_t`, `gclient_t`, and `level_locals_t` from `g_local.h`.

## Global / File-Static State
None.

## Key Functions / Methods

### TeleportPlayer
- **Signature:** `void TeleportPlayer( gentity_t *player, vec3_t origin, vec3_t angles )`
- **Purpose:** Instantly moves a player entity to a new origin and orientation, applying an exit velocity and suppressing client-side interpolation.
- **Inputs:** `player` — the entity to teleport; `origin` — destination world position; `angles` — destination facing angles.
- **Outputs/Return:** void
- **Side effects:** Spawns two temp events (`EV_PLAYER_TELEPORT_OUT`, `EV_PLAYER_TELEPORT_IN`); mutates `player->client->ps` (origin, velocity, pm_time, pm_flags, eFlags); calls `G_KillBox` to destroy anything at destination; re-links entity; updates `player->r.currentOrigin` and entity state via `BG_PlayerStateToEntityState`.
- **Calls:** `G_TempEntity`, `trap_UnlinkEntity`, `AngleVectors`, `VectorScale`, `SetClientViewAngle`, `G_KillBox`, `BG_PlayerStateToEntityState`, `trap_LinkEntity`
- **Notes:** Spectators skip teleport effects and `G_KillBox`. The `EF_TELEPORT_BIT` toggle prevents lerp artifacts on the client.

### locateCamera
- **Signature:** `void locateCamera( gentity_t *ent )`
- **Purpose:** Deferred think function that resolves a `misc_portal_surface`'s target camera entity and configures the portal entity's state fields for renderer consumption.
- **Inputs:** `ent` — the portal surface entity.
- **Outputs/Return:** void
- **Side effects:** Sets `ent->r.ownerNum`, `ent->s.frame`, `ent->s.powerups`, `ent->s.clientNum`, `ent->s.origin2`, `ent->s.eventParm`; may free `ent` if target not found.
- **Calls:** `G_PickTarget`, `G_Printf`, `G_FreeEntity`, `VectorSubtract`, `VectorNormalize`, `G_SetMovedir`, `DirToByte`
- **Notes:** Called via `think` pointer 100 ms after spawn to allow targets to be spawned first.

### Use_Shooter
- **Signature:** `void Use_Shooter( gentity_t *ent, gentity_t *other, gentity_t *activator )`
- **Purpose:** `use` callback for shooter entities; fires the configured projectile toward a target or along `movedir`, with random angular deviation.
- **Inputs:** `ent` — shooter entity; `other`, `activator` — standard use chain (unused here).
- **Outputs/Return:** void
- **Side effects:** Spawns a projectile entity; fires `EV_FIRE_WEAPON` event on `ent`.
- **Calls:** `PerpendicularVector`, `CrossProduct`, `crandom`, `VectorMA`, `VectorNormalize`, `fire_grenade` / `fire_rocket` / `fire_plasma`, `G_AddEvent`
- **Notes:** Dispatches on `ent->s.weapon`; only three weapon types handled.

### InitShooter
- **Signature:** `void InitShooter( gentity_t *ent, int weapon )`
- **Purpose:** Common initialization for all `shooter_*` entity variants.
- **Inputs:** `ent` — the shooter entity; `weapon` — `WP_*` constant.
- **Side effects:** Sets `use`, `s.weapon`, `movedir`; registers the item; optionally defers target resolution via `InitShooter_Finish`; links entity.
- **Calls:** `RegisterItem`, `BG_FindItemForWeapon`, `G_SetMovedir`, `trap_LinkEntity`

### DropPortalDestination / DropPortalSource (MISSIONPACK)
- **Purpose:** Create dynamic portal destination and source entities at a player's current position when they use the portal holdable item; link them by `portalID`.
- **Side effects:** Spawns new `gentity_t`s, sets `level.portalSequence`, mutates `player->client->portalID` and `ps.stats[STAT_HOLDABLE_ITEM]`.
- **Calls:** `G_Spawn`, `G_ModelIndex`, `G_SetOrigin`, `trap_LinkEntity`, `G_FreeEntity` (as think), `BG_FindItem`

## Control Flow Notes
- All `SP_*` functions are spawn callbacks invoked by `G_SpawnEntitiesFromString` during map load.
- `TeleportPlayer` is called at runtime from `g_trigger.c` (`trigger_teleporter_touch`) and from `PortalTouch` (MISSIONPACK).
- `locateCamera` is a deferred think scheduled 100 ms post-spawn to guarantee target entities exist.
- Shooter entities fire only when triggered via their `use` callback; no per-frame update logic.

## External Dependencies
- **`g_local.h`** — pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, all `gentity_t`/`gclient_t` definitions, and all `trap_*` syscall declarations.
- **Defined elsewhere:** `G_TempEntity`, `G_KillBox`, `G_PickTarget`, `G_SetMovedir`, `BG_PlayerStateToEntityState`, `SetClientViewAngle`, `fire_grenade`, `fire_rocket`, `fire_plasma`, `RegisterItem`, `BG_FindItemForWeapon`, `Drop_Item`, `BG_FindItemForPowerup`, `G_Damage`, `G_Find`, `G_Spawn`, `G_SetOrigin`, `G_FreeEntity`, `DirToByte`, `PerpendicularVector`, `CrossProduct`, `crandom`, `level` (global), all `trap_*` functions.
