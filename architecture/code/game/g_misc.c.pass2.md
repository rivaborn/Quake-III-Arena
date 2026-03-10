# code/game/g_misc.c — Enhanced Analysis

## Architectural Role

This file is a utility module within the **Game VM** (server-side game logic) that provides spawn callbacks for miscellaneous map entities and implements core gameplay mechanics for teleportation, portal rendering, and trigger-based weapon firing. It bridges the map editor's entity definitions to runtime authoritative game state, operating at the intersection of level design and gameplay physics—all within the sandboxed QVM bytecode execution environment.

## Key Cross-References

### Incoming (who depends on this file)
- **`g_spawn.c` dispatcher**: All `SP_*` functions are registered in a spawn function table and called once per entity during map load (`G_SpawnEntitiesFromString`).
- **`g_trigger.c`**: `TeleportPlayer` is called by `trigger_teleporter_touch` when a player touches a teleporter trigger; also called from MISSIONPACK portal touch handlers in this file itself.
- **`g_utils.c` / `g_main.c`**: `G_PickTarget` resolution is called at runtime; entities may reference `TeleportPlayer` indirectly through trigger systems.
- **cgame VM** (client-side): Receives the rendered results of portal surface state via the `ET_PORTAL` entity type and portal camera properties encoded in `s.origin2`, `s.frame`, `s.powerups`, etc.
- **Server snapshots**: Portal and shooter entity state is delta-compressed and sent to clients; client-side rendering of portal surfaces depends on the data fields set by `locateCamera`.

### Outgoing (what this file depends on)
- **Game module internals**: `G_TempEntity` (spawns temporary events), `G_KillBox` (destroys contents), `G_SetOrigin` (position setter), `G_FreeEntity` (destruction), `G_PickTarget` (target resolution), `G_SetMovedir` (angle-to-direction), `G_Spawn` (entity allocation).
- **Engine trap layer** (`trap_*` syscalls):
  - `trap_LinkEntity` / `trap_UnlinkEntity` — spatial hashing for collision and snapshot visibility
  - `trap_ModelIndex` — resolve asset strings to renderer indices
  - `trap_R_AddLightToScene` — (indirectly via trap dispatcher)
- **Background (shared) layer** (`bg_*`):
  - `BG_PlayerStateToEntityState` — synchronize authoritative player state to networked entity state
  - `BG_FindItemForWeapon` / `BG_FindItemForPowerup` / `BG_FindItem` — item database lookup (used for weapon registration and powerup drops)
- **Physics/math**: `AngleVectors`, `VectorScale`, `VectorCopy`, `VectorNormalize`, `VectorSubtract`, `PerpendicularVector`, `CrossProduct`, `VectorMA`, `crandom`, `DirToByte` — all utility functions from `q_math.c` / `q_shared.c`.
- **Weapon systems**: `fire_grenade`, `fire_rocket`, `fire_plasma` — projectile spawning functions (likely in `g_weapon.c` or `g_missile.c`).
- **Event system**: `G_AddEvent` — queues networked events (e.g., `EV_FIRE_WEAPON`, `EV_PLAYER_TELEPORT_OUT`, `EV_PLAYER_TELEPORT_IN`) for cgame consumption.
- **Client/player state**: Direct manipulation of `player->client->ps` (player state) and `player->client->sess` (session data).

## Design Patterns & Rationale

### Spawn Callback Convention
All `SP_*` functions follow a stereotyped contract: `void SP_name(gentity_t *ent)`. This allows `g_spawn.c`'s dispatcher to invoke them polymorphically based on the entity's `classname` field. Some entities (e.g., `info_null`, `light`, `misc_model`) are destroyed immediately (`G_FreeEntity`) because they are editor-only or have no runtime role.

### Deferred Initialization via Think Callback
`locateCamera` is scheduled 100 ms post-spawn (`ent->nextthink = level.time + 100`) rather than executing immediately. This pattern ensures all map entities have been spawned before target resolution occurs—a **simple but effective temporal decoupling** that avoids fragile ordering dependencies.

### Use Callback Pattern
`Use_Shooter` is assigned as `ent->use` during `InitShooter`, making the shooter entity trigger-responsive. This is idiomatic to Q3A's event-driven entity system: entities define callbacks for distinct lifecycle events (think, use, touch, die) rather than a monolithic per-frame update.

### Portal ID Matching (MISSIONPACK)
The portal system links source and destination entities via a shared `count` field initialized from `player->client->portalID`. This is a **stateful, runtime-assigned identity** that couples the source and destination at creation time—simple but vulnerable to ID collisions if portal entities overlap in creation (mitigated by sequential `level.portalSequence` increment).

## Data Flow Through This File

### Teleportation Flow
1. **Trigger activation**: A player touches a `trigger_teleporter` → calls `TeleportPlayer(player, dest_origin, dest_angles)`.
2. **State mutation**:
   - Spawn temp events (`EV_PLAYER_TELEPORT_OUT` at source, `EV_PLAYER_TELEPORT_IN` at destination) so cgame renders exit/entry effects.
   - Unlink from spatial hash, copy new origin/angles to `player->client->ps`.
   - Apply exit velocity via `AngleVectors(angles)` + `VectorScale(..., 400, ...)`.
   - Set `EF_TELEPORT_BIT` toggle to signal cgame to suppress client-side interpolation (prevent lerp artifacts).
   - Call `G_KillBox` to instantly destroy anything at destination (prevents telefragging).
3. **Synchronization**: `BG_PlayerStateToEntityState` copies the mutated `ps` to the network-visible `s`, and entity re-links to spatial hash.
4. **Next snapshot**: Changes are delta-compressed and sent to all clients; cgame consumes the teleport event and updates client prediction.

### Portal System Flow (MISSIONPACK)
1. **Player drops source**: `DropPortalSource` → spawns `hi_portal source` entity at player location, defers `PortalEnable` think 1 s later, resolves matching destination and stores its origin in `ent->pos1` (fallback teleport destination if source is destroyed).
2. **Portal source enabled** (1 s delay): `PortalEnable` sets `self->touch = PortalTouch` and schedules destruction in 2 minutes.
3. **Player touches portal**: `PortalTouch` → drops any flags, searches for matching `hi_portal destination` by ID, teleports player via `TeleportPlayer` or applies fallback damage.
4. **Lifecycle**: Both source and destination are destroyed after 2 minutes by `G_FreeEntity` think callback, or by `PortalDie` if damaged.

### Shooter Fire Flow
1. **Initialization** (`InitShooter`): Register weapon item, compute `movedir` from angles, schedule `InitShooter_Finish` if target specified.
2. **Deferred target resolution** (500 ms later): `InitShooter_Finish` resolves `ent->enemy` from `G_PickTarget(ent->target)`.
3. **Trigger activation** (`Use_Shooter`): Compute firing direction (target or movedir with random angular deviation), dispatch to weapon-specific fire function, queue `EV_FIRE_WEAPON` event.

### Portal Surface Rendering (client-side consumer)
1. **Initialization** (`SP_misc_portal_surface`): Set `ET_PORTAL` type, set `SVF_PORTAL` server flag, schedule `locateCamera` think 100 ms later.
2. **Camera resolution** (`locateCamera`): Resolve target camera entity, encode rotation speed in `s.frame`, encode camera direction in `s.eventParm` (via `DirToByte`), set `s.origin2` to camera position.
3. **Client-side** (in cgame/renderer): Renderer reads portal entity state, resolves the target camera from `r.ownerNum`, renders an off-screen view from the camera's perspective, and composites it as a mirror/portal overlay on the nearest surface within 64 units.

## Learning Notes

### Q3A-Era Design Idioms
- **Entity-as-data-container + callback dispatch**: Rather than a class hierarchy, entities are bags of data with function pointers (`use`, `think`, `touch`, `die`). This is lightweight, cache-hostile by modern standards, but was pragmatic for late-1990s engines and allows polymorphism without vtables.
- **Deferred initialization via think pointer**: Avoids ordering constraints and keeps init code localized. Modern engines often use dependency injection or explicit initialization phases.
- **Networked state synchronization via delta-compression**: The engine never sends full entity state; only fields that changed are sent, reducing bandwidth. This required tight coupling between `playerState_t`/`entityState_t` definitions and both game and cgame VMs.

### Separation of Concerns: Server vs. Client
- **g_misc.c** (server): Owns all state transitions and authoritative logic. Portal IDs, teleport destinations, and shooter firing are all server-side.
- **cgame VM** (client): Consumes events and state to render effects. The portal rendering, teleport FX, and weapon fire events are purely visual/informational.
- This design prevents clients from cheating (e.g., teleporting without server approval) but requires careful event/state synchronization.

### Spawn Function Dispatch Pattern
The `SP_*` convention is a form of **name-based polymorphism**: the level editor stores entity `classname` strings, and `g_spawn.c` maintains a static table mapping classnames to function pointers. This is less flexible than dynamic registration but avoids runtime symbol lookup overhead.

### Physics Integration
`TeleportPlayer` directly mutates `player->client->ps` (player state) and relies on the engine's next `Pmove` tick to validate the new position. There's a subtle race: if the player is out-of-bounds after teleport, the next `Pmove` will attempt corrections (clipmodel adjustment, ground plane recalculation), which may cause snapping or stuttering. The code assumes destination coordinates are always valid (enforced by level design).

## Potential Issues

1. **Spectator Skip Inconsistency** (lines 90–119): Spectators skip temp events and `G_KillBox` during teleport. While intentional (spectators phase through everything), this creates behavioral divergence that could surprise modders or cause bugs if spectator handling is later extended.

2. **Unvalidated Camera Target** (line 159): `locateCamera` calls `G_PickTarget(owner->target)` but does not validate that `target` exists; if it fails, `dir` is left uninitialized. This is masked by the null-pointer check after `G_PickTarget`, but the initialization sequence is fragile.

3. **Portal ID Collision** (MISSIONPACK, line 355): Sequential assignment of portal IDs via `++level.portalSequence` assumes IDs never wrap or collide. With 32-bit integers and millions of portal drops per map, wrapping is theoretically possible and would cause old portal pairs to mysteriously re-link.

4. **Fallback Destination Hard-Coded** (line 331): If a portal source's destination is destroyed but the source persists beyond 2 minutes (time-extended manually), falling into it applies 100k damage and teleports to `ent->pos1`. The fallback is silent (no event) and could leave players stranded if `pos1` is invalid.

5. **Shooter Random Deviation Unclamped** (line 247): The `random` parameter is converted to radians via `sin(M_PI * random / 180)`, but there's no clamp on `random` input. Values >180 produce negative deviation; this is likely intentional but not documented.

6. **No Portal Source Reachability Check** (line 374): `DropPortalSource` never validates that the destination is actually reachable or on a valid floor. If dropped mid-air or in invalid geometry, the destination entity may be unreachable by normal pathfinding.
