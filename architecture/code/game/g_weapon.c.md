# code/game/g_weapon.c

## File Purpose
Implements all server-side weapon firing logic for Quake III Arena, translating player weapon inputs into world-space traces, damage events, and projectile spawns. It is the authoritative damage source for hitscan weapons and the launch point for projectile entities.

## Core Responsibilities
- Compute muzzle position and firing direction from player view state
- Execute hitscan traces for gauntlet, machinegun, shotgun, railgun, and lightning gun
- Spawn projectile entities for rocket, grenade, plasma, BFG, grapple (and MissionPack: nail, prox mine)
- Apply Quad Damage (and MISSIONPACK Doubler) multipliers to all outgoing damage
- Track per-client shot/hit accuracy counters; award "Impressive" for back-to-back railgun hits
- Emit temp entities (EV_BULLET_HIT_FLESH, EV_RAILTRAIL, EV_SHOTGUN, etc.) for client-side effects
- MISSIONPACK: handle Kamikaze holdable item with expanding radius damage and shockwave

## Key Types / Data Structures
None (file operates on types defined in `g_local.h`/`bg_public.h`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_quadFactor` | `float` | static (file) | Current damage multiplier (1.0, Quad, or Doubler); set in `FireWeapon` / `CheckGauntletAttack` |
| `forward` | `vec3_t` | static (file) | Firing-direction unit vector derived from player viewangles |
| `right` | `vec3_t` | static (file) | Right vector, used for spread calculations |
| `up` | `vec3_t` | static (file) | Up vector, used for spread calculations |
| `muzzle` | `vec3_t` | static (file) | World-space muzzle origin, set once per `FireWeapon` call |

## Key Functions / Methods

### FireWeapon
- **Signature:** `void FireWeapon( gentity_t *ent )`
- **Purpose:** Top-level entry point called once per weapon fire event. Sets `s_quadFactor`, computes aim vectors and muzzle point, then dispatches to the appropriate per-weapon function.
- **Inputs:** `ent` — firing player entity
- **Outputs/Return:** void
- **Side effects:** Writes `s_quadFactor`, `forward`, `right`, `up`, `muzzle` (all file-static); increments `ent->client->accuracy_shots`
- **Calls:** `AngleVectors`, `CalcMuzzlePointOrigin`, per-weapon fire functions
- **Notes:** Grapple and gauntlet do not increment `accuracy_shots`. Team play uses a weaker machinegun damage value.

### CheckGauntletAttack
- **Signature:** `qboolean CheckGauntletAttack( gentity_t *ent )`
- **Purpose:** Called each frame a player holds gauntlet fire; performs a 32-unit forward trace and applies melee damage.
- **Inputs:** `ent` — attacking player
- **Outputs/Return:** `qtrue` if a damageable entity was hit
- **Side effects:** Sets `s_quadFactor`; calls `G_Damage`; spawns `EV_MISSILE_HIT` temp entity
- **Calls:** `AngleVectors`, `CalcMuzzlePoint`, `trap_Trace`, `G_TempEntity`, `G_AddEvent`, `G_Damage`

### Bullet_Fire
- **Signature:** `void Bullet_Fire( gentity_t *ent, float spread, int damage )`
- **Purpose:** Fires a single hitscan bullet with randomized angular spread; handles MISSIONPACK invulnerability bouncing.
- **Inputs:** `ent` — shooter; `spread` — half-angle in Q3 units; `damage` — base damage (pre-quad)
- **Outputs/Return:** void
- **Side effects:** Spawns EV_BULLET_HIT_FLESH or EV_BULLET_HIT_WALL temp entity; calls `G_Damage`; increments `accuracy_hits`
- **Calls:** `trap_Trace`, `SnapVectorTowards`, `G_TempEntity`, `LogAccuracyHit`, `G_Damage`, (MP) `G_InvulnerabilityEffect`, `G_BounceProjectile`

### weapon_railgun_fire
- **Signature:** `void weapon_railgun_fire( gentity_t *ent )`
- **Purpose:** Fires railgun; iteratively unlinks hit entities so the beam penetrates multiple targets up to `MAX_RAIL_HITS` (4). Awards "Impressive" on 2 consecutive hits.
- **Inputs:** `ent` — shooter
- **Outputs/Return:** void
- **Side effects:** Calls `G_Damage` on all penetrated entities; spawns `EV_RAILTRAIL` temp entity; modifies `ent->client->accurateCount`, `PERS_IMPRESSIVE_COUNT`, `eFlags`
- **Calls:** `trap_Trace`, `trap_UnlinkEntity`, `trap_LinkEntity`, `G_Damage`, `LogAccuracyHit`, `G_TempEntity`, `SnapVectorTowards`

### Weapon_LightningFire
- **Signature:** `void Weapon_LightningFire( gentity_t *ent )`
- **Purpose:** Fires lightning gun up to `LIGHTNING_RANGE`; MISSIONPACK variant can bounce off invulnerability spheres across multiple iterations.
- **Inputs:** `ent` — shooter
- **Outputs/Return:** void
- **Side effects:** `G_Damage`; spawns EV_MISSILE_HIT or EV_MISSILE_MISS temp entities; increments `accuracy_hits`
- **Calls:** `trap_Trace`, `G_Damage`, `G_TempEntity`, `LogAccuracyHit`, (MP) `G_InvulnerabilityEffect`, `G_BounceProjectile`

### ShotgunPattern
- **Signature:** `void ShotgunPattern( vec3_t origin, vec3_t origin2, int seed, gentity_t *ent )`
- **Purpose:** Generates `DEFAULT_SHOTGUN_COUNT` pellets using a seeded PRNG matching the client-side prediction in `CG_ShotgunPattern`.
- **Inputs:** `origin` — muzzle pos; `origin2` — encoded forward direction; `seed` — spread seed; `ent` — shooter
- **Side effects:** Calls `ShotgunPellet` for each pellet; increments `accuracy_hits` once if any pellet hits a client
- **Calls:** `VectorNormalize2`, `PerpendicularVector`, `CrossProduct`, `Q_crandom`, `ShotgunPellet`
- **Notes:** Seed must match client to keep visual spread consistent with server-authoritative damage.

### G_StartKamikaze *(MISSIONPACK)*
- **Signature:** `void G_StartKamikaze( gentity_t *ent )`
- **Purpose:** Spawns a Kamikaze explosion entity that expands radius damage and shockwave over ~1 second via `KamikazeDamage` think function.
- **Inputs:** `ent` — entity that activated kamikaze
- **Side effects:** Spawns explosion `gentity_t`; deals 100,000 damage to activating player; broadcasts `EV_GLOBAL_TEAM_SOUND` / `GTS_KAMIKAZE`
- **Calls:** `G_Spawn`, `G_SetOrigin`, `trap_LinkEntity`, `G_Damage`, `G_TempEntity`

### CalcMuzzlePoint / CalcMuzzlePointOrigin
- **Purpose:** Compute the world-space muzzle position from the player's base position + viewheight + 14 units forward, then snap to integers for bandwidth.
- **Notes:** `CalcMuzzlePointOrigin` accepts an explicit `origin` parameter but does not use it (identical body to `CalcMuzzlePoint`); the `origin` parameter appears vestigial.

### LogAccuracyHit
- **Signature:** `qboolean LogAccuracyHit( gentity_t *target, gentity_t *attacker )`
- **Purpose:** Returns `qtrue` only when a hit should count toward accuracy (target is a living, enemy client).
- **Notes:** Filters out self-hits, team hits, non-client targets, and dead targets.

### G_BounceProjectile
- **Signature:** `void G_BounceProjectile( vec3_t start, vec3_t impact, vec3_t dir, vec3_t endout )`
- **Purpose:** Reflects a hitscan ray off a surface normal; used for MISSIONPACK invulnerability sphere interactions.

## Control Flow Notes
`FireWeapon` is called from `g_active.c` (`ClientThink`) when the player's weapon fires. It runs once per fire event (not per frame for non-automatic weapons). The file-static vectors (`forward`, `right`, `up`, `muzzle`, `s_quadFactor`) are written at the top of `FireWeapon` and remain valid for the duration of all sub-calls within that single firing event. `Weapon_HookThink` is registered as a per-frame think callback on the grapple hook entity.

## External Dependencies
- `g_local.h` → `q_shared.h`, `bg_public.h`, `g_public.h` (all game types and trap declarations)
- **Defined elsewhere:** `g_entities[]`, `level` (globals in `g_main.c`); `fire_rocket`, `fire_grenade`, `fire_plasma`, `fire_bfg`, `fire_grapple`, `fire_nail`, `fire_prox` (`g_missile.c`); `G_Damage`, `G_InvulnerabilityEffect` (`g_combat.c`); `OnSameTeam` (`g_team.c`); `g_quadfactor`, `g_gametype` (cvars registered in `g_main.c`); `trap_Trace`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_EntitiesInBox` (engine syscalls)
