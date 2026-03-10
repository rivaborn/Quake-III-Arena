# code/game/g_missile.c

## File Purpose
Implements server-side missile entity creation, movement simulation, and impact handling for all projectile weapons in Quake III Arena. It spawns missile entities, advances them each frame via trajectory evaluation and collision tracing, and dispatches bounce, impact, or explosion logic on collision.

## Core Responsibilities
- Spawn typed missile entities (plasma, grenade, rocket, BFG, grapple, and MISSIONPACK: nail, prox mine)
- Advance missiles each server frame: evaluate trajectory, trace movement, detect collisions
- Handle missile impact: apply direct damage, splash damage, bounce, grapple attachment
- Manage MISSIONPACK proximity mine lifecycle: activation, trigger volumes, player-sticking, timed explosion
- Emit network events (hit/miss/bounce/explosion) for client-side effects
- Track accuracy hits on the owning client

## Key Types / Data Structures
None defined in this file; relies entirely on types from `g_local.h`.

| Name | Kind | Purpose |
|---|---|---|
| `gentity_t` | struct (extern) | Game entity; missiles are gentity_t instances with weapon-specific fields |
| `trace_t` | struct (extern) | Collision trace result used for impact detection and bounce math |
| `level_locals_t` (`level`) | struct (extern global) | Provides `level.time`, `level.previousTime` for trajectory and timing |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `MISSILE_PRESTEP_TIME` | `#define` (50 ms) | file | Back-dates `trTime` so missiles move slightly on their first frame |

## Key Functions / Methods

### G_BounceMissile
- **Signature:** `void G_BounceMissile( gentity_t *ent, trace_t *trace )`
- **Purpose:** Reflects missile velocity off the impact plane; optionally damps and stops it.
- **Inputs:** Missile entity, collision trace with plane normal and fraction.
- **Outputs/Return:** void; mutates `ent->s.pos.trDelta`, `trBase`, `trTime`, `r.currentOrigin`.
- **Side effects:** May call `G_SetOrigin` to freeze a stopped `EF_BOUNCE_HALF` entity.
- **Calls:** `BG_EvaluateTrajectoryDelta`, `DotProduct`, `VectorMA`, `VectorScale`, `VectorLength`, `G_SetOrigin`, `VectorAdd`, `VectorCopy`.
- **Notes:** Interpolates hit time between `level.previousTime` and `level.time` via `trace->fraction` for accuracy.

### G_ExplodeMissile
- **Signature:** `void G_ExplodeMissile( gentity_t *ent )`
- **Purpose:** Detonates a missile in-place (timeout or indirect trigger) without a surface impact.
- **Inputs:** Missile entity.
- **Outputs/Return:** void.
- **Side effects:** Fires `EV_MISSILE_MISS` event; applies splash damage; increments `accuracy_hits` on owner; sets `freeAfterEvent`; calls `trap_LinkEntity`.
- **Calls:** `BG_EvaluateTrajectory`, `SnapVector`, `G_SetOrigin`, `G_AddEvent`, `G_RadiusDamage`, `trap_LinkEntity`.

### G_MissileImpact
- **Signature:** `void G_MissileImpact( gentity_t *ent, trace_t *trace )`
- **Purpose:** Central impact dispatcher: applies direct damage, splash damage, fires hit/miss events, handles grapple attachment, bounce, prox mine sticking, and invulnerability deflection.
- **Inputs:** Missile entity, collision trace.
- **Outputs/Return:** void.
- **Side effects:** Modifies target entity health via `G_Damage`/`G_RadiusDamage`; spawns new entities (grapple, prox trigger); emits network events; sets `freeAfterEvent`; calls `trap_LinkEntity`.
- **Calls:** `G_BounceMissile`, `G_AddEvent`, `G_Damage`, `G_RadiusDamage`, `LogAccuracyHit`, `BG_EvaluateTrajectoryDelta`, `G_Spawn`, `G_SetOrigin`, `SnapVectorTowards`, `trap_LinkEntity`, `G_InvulnerabilityEffect` (MISSIONPACK), `ProximityMine_Player` (MISSIONPACK), `Weapon_HookThink`.
- **Notes:** Sky surfaces (`SURF_NOIMPACT`) are handled upstream in `G_RunMissile`, not here. The grapple hook spawns a separate visual-only `nent` at impact point and converts `ent` to `ET_GRAPPLE`.

### G_RunMissile
- **Signature:** `void G_RunMissile( gentity_t *ent )`
- **Purpose:** Per-frame update: evaluates trajectory, sweeps a collision trace, calls impact or think logic.
- **Inputs:** Missile entity.
- **Outputs/Return:** void.
- **Side effects:** Mutates `ent->r.currentOrigin`; frees entity on sky hit; calls `trap_Trace`, `trap_LinkEntity`, `G_MissileImpact`, `G_RunThink`.
- **Calls:** `BG_EvaluateTrajectory`, `trap_Trace`, `trap_LinkEntity`, `G_FreeEntity`, `G_MissileImpact`, `G_RunThink`.
- **Notes:** `passent` is set to `ent->r.ownerNum` by default so the missile ignores its owner. MISSIONPACK: prox mines use `ENTITYNUM_NONE` once outside the owner's bbox (`ent->count == 1`).

### fire_plasma / fire_grenade / fire_bfg / fire_rocket / fire_grapple
- **Signature pattern:** `gentity_t *fire_X( gentity_t *self, vec3_t start, vec3_t dir )`
- **Purpose:** Factory functions; allocate and initialize a missile entity with weapon-specific parameters and trajectory.
- **Inputs:** Firing entity, world-space muzzle origin, normalized direction.
- **Outputs/Return:** Pointer to the new missile `gentity_t`.
- **Side effects:** Calls `G_Spawn`; `fire_grapple` additionally sets `self->client->hook`.
- **Notes:** All use `level.time - MISSILE_PRESTEP_TIME` as `trTime` to avoid a stationary first frame. Speed constants: plasma/BFG 2000, rocket 900, grenade/grapple 700–800 ups.

### ProximityMine_* (MISSIONPACK only)
- **Notes:** A cluster of static helper functions managing the prox mine state machine: `ProximityMine_Activate` (arm after 2 s, build trigger volume), `ProximityMine_Trigger` (touch callback — sphere-distance check, team check, LOS check), `ProximityMine_Player` (stick to player, share damage with stacked mines), `ProximityMine_ExplodeOnPlayer` (timed detonation on stuck player), `ProximityMine_Die`/`ProximityMine_Explode` (damage-induced detonation).

## Control Flow Notes
`G_RunMissile` is called every server frame for each `ET_MISSILE` entity from `G_RunThink` dispatch in `g_main.c`. The `fire_*` functions are called from `g_weapon.c:FireWeapon` at the moment of firing. Impact handling is entirely synchronous within the frame.

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — `bg_pmove.c`/`bg_misc.c`
  - `G_Damage`, `G_RadiusDamage`, `CanDamage`, `G_InvulnerabilityEffect` — `g_combat.c`
  - `LogAccuracyHit`, `Weapon_HookFree`, `Weapon_HookThink`, `SnapVectorTowards` — `g_weapon.c`
  - `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `G_AddEvent`, `G_SoundIndex` — `g_utils.c`
  - `G_RunThink` — `g_main.c`
  - `trap_Trace`, `trap_LinkEntity` — engine syscall stubs (`g_syscalls.c`)
  - `level`, `g_entities`, `g_proxMineTimeout`, `g_gametype` — game module globals
