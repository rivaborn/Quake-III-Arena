# code/game/g_combat.c

## File Purpose
Implements all server-side combat logic for Quake III Arena's game module, including damage application, knockback, scoring, death processing, item drops, and radius explosion damage. It serves as the central damage pipeline that all weapons and hazards funnel through.

## Core Responsibilities
- Apply damage to entities via `G_Damage`, handling armor absorption, knockback, godmode, team protection, and invulnerability
- Execute player death sequence via `player_die`, including obituary logging, scoring, animation, and flag/item handling
- Perform area-of-effect damage via `G_RadiusDamage` with line-of-sight gating
- Drop held weapons and powerups on player death via `TossClientItems`
- Manage score additions and visual score plums via `AddScore`/`ScorePlum`
- Handle gib deaths and body corpse state transitions via `GibEntity`/`body_die`
- Detect near-capture/near-score events for "holy shit" reward triggers

## Key Types / Data Structures
None defined in this file; all types come from `g_local.h`.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `modNames[]` | `char *[]` | file-static | String table mapping `meansOfDeath` enum values to log strings |
| `neutralObelisk` | `gentity_t *` (extern) | global (defined elsewhere) | Reference to the neutral obelisk entity used in Harvester cube toss |
| `i` (in `player_die`) | `static int` | function-static | Cycles through 3 death animations round-robin |

## Key Functions / Methods

### G_Damage
- **Signature:** `void G_Damage(gentity_t *targ, gentity_t *inflictor, gentity_t *attacker, vec3_t dir, vec3_t point, int damage, int dflags, int mod)`
- **Purpose:** Central damage application function; routes all weapon/environmental damage to entities.
- **Inputs:** Target, inflictor, attacker entities; direction/point vectors; damage amount; damage flags (`DAMAGE_RADIUS`, `DAMAGE_NO_ARMOR`, etc.); means of death.
- **Outputs/Return:** void
- **Side effects:** Modifies `targ->health`, `targ->client->ps` velocity/stats, `client->damage_*` accumulation fields; calls `targ->die()` or `targ->pain()`; triggers `EV_POWERUP_BATTLESUIT` event.
- **Calls:** `G_InvulnerabilityEffect`, `CheckObeliskAttack`, `CheckArmor`, `OnSameTeam`, `G_AddEvent`, `LogAccuracyHit`, `Team_CheckHurtCarrier`, `G_Printf`, `targ->die`, `targ->pain`
- **Notes:** Skips damage during `level.intermissionQueued`; applies attacker handicap scaling; self-damage is halved; minimum 1 damage enforced; noclip clients are immune.

### player_die
- **Signature:** `void player_die(gentity_t *self, gentity_t *inflictor, gentity_t *attacker, int damage, int meansOfDeath)`
- **Purpose:** Full player death sequence: scoring, obituary broadcast, animation, corpse setup, item drops, flag returns.
- **Inputs:** Dying player, inflictor, attacker, damage dealt, cause of death.
- **Outputs/Return:** void
- **Side effects:** Sets `PM_DEAD`, logs kill, broadcasts `EV_OBITUARY` temp entity, modifies scores, drops items, sets `respawnTime`, clears powerups, links entity; triggers gib or death animation.
- **Calls:** `CheckAlmostCapture`, `CheckAlmostScored`, `Weapon_HookFree`, `AddScore`, `Team_FragBonuses`, `Team_ReturnFlag`, `TossClientItems`, `TossClientPersistantPowerups`, `TossClientCubes`, `Cmd_Score_f`, `GibEntity`, `G_AddEvent`, `LookAtKiller`, `Kamikaze_DeathTimer`, `trap_LinkEntity`, `G_TempEntity`, `G_LogPrintf`
- **Notes:** Guards against re-entry via `PM_DEAD` check; death animation cycles via a `static int i` with round-robin modulo 3; gauntlet kills and rapid multi-kills grant award `eFlags`.

### G_RadiusDamage
- **Signature:** `qboolean G_RadiusDamage(vec3_t origin, gentity_t *attacker, float damage, float radius, gentity_t *ignore, int mod)`
- **Purpose:** Applies falloff damage to all damageable entities within a sphere, gated by `CanDamage` LOS check.
- **Inputs:** Explosion origin, attacker, max damage, radius, entity to ignore, means of death.
- **Outputs/Return:** `qtrue` if any client was hit (for accuracy tracking).
- **Side effects:** Calls `G_Damage` for each entity in range; modifies entity velocities via knockback.
- **Calls:** `trap_EntitiesInBox`, `VectorLength`, `CanDamage`, `LogAccuracyHit`, `G_Damage`
- **Notes:** Damage scales linearly from full at center to zero at radius edge; direction vector has `dir[2] += 24` to bias knockback upward.

### CanDamage
- **Signature:** `qboolean CanDamage(gentity_t *targ, vec3_t origin)`
- **Purpose:** LOS check for explosion damage; fires up to 5 traces to the target bounding box corners.
- **Calls:** `trap_Trace`
- **Notes:** Uses bounding box midpoint and four ±15-unit offsets in XY plane; does not check Z offsets.

### CheckArmor
- **Signature:** `int CheckArmor(gentity_t *ent, int damage, int dflags)`
- **Purpose:** Computes and deducts armor absorption from a damage hit.
- **Outputs/Return:** Amount of damage absorbed by armor.
- **Notes:** Absorption is `ceil(damage * ARMOR_PROTECTION)`; clamps to available armor.

### TossClientItems
- **Signature:** `void TossClientItems(gentity_t *self)`
- **Purpose:** Drops held weapon and active powerups as world items on player death.
- **Notes:** MG and grapple hook are never dropped; weapon-change-in-progress is resolved; powerup time-remaining is stored in `drop->count`; drops are skipped in team games for powerups.

### Notes (minor functions)
- `ScorePlum`: Creates a single-client temp entity for the floating score display.
- `AddScore`: Guards warmup, calls `ScorePlum`, updates `PERS_SCORE` and team scores, then `CalculateRanks`.
- `LookAtKiller`: Sets `STAT_DEAD_YAW` so corpse faces its killer.
- `GibEntity`: Fires `EV_GIB_PLAYER`, clears `takedamage`, sets `ET_INVISIBLE` and zero contents; handles kamikaze timer cleanup.
- `body_die`: Corpse gib callback; re-gibs if health drops below `GIB_HEALTH` and blood is enabled.
- `RaySphereIntersections`: Quadratic ray-sphere solver used by `G_InvulnerabilityEffect`.
- `G_InvulnerabilityEffect` (MISSIONPACK): Spawns `EV_INVUL_IMPACT` temp entity and returns bounce direction for deflected projectiles.
- `CheckAlmostCapture` / `CheckAlmostScored`: Toggle `PLAYEREVENT_HOLYSHIT` flag if a flag/cube carrier is killed within 200 units of the objective.
- `Kamikaze_DeathTimer` / `Kamikaze_DeathActivate` (MISSIONPACK): Deferred 5-second kamikaze explosion on carrier death.
- `TossClientCubes` / `TossClientPersistantPowerups` (MISSIONPACK): Harvester cube and persistent powerup drop logic.

## Control Flow Notes
`g_combat.c` is event-driven within the game frame. `G_Damage` is called from missile impact handlers (`g_missile.c`), weapon fire (`g_weapon.c`), trigger volumes (`g_trigger.c`), and mover crush logic. It is not called directly from the frame loop; rather it is invoked as a callback from entity logic. `player_die` is installed as the `->die` function pointer on client entities during `ClientSpawn`.

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `g_entities[]`, `level` — global game state (`g_main.c`)
  - `g_knockback`, `g_blood`, `g_friendlyFire`, `g_gametype`, `g_debugDamage`, `g_cubeTimeout` — cvars
  - `Team_FragBonuses`, `Team_ReturnFlag`, `Team_CheckHurtCarrier`, `OnSameTeam` — `g_team.c`
  - `Drop_Item`, `LaunchItem`, `BG_FindItemForWeapon`, `BG_FindItemForPowerup`, `BG_FindItem` — items/bg layer
  - `Weapon_HookFree`, `LogAccuracyHit` — `g_weapon.c`
  - `Cmd_Score_f` — `g_cmds.c`
  - `G_StartKamikaze` — `g_weapon.c` (MISSIONPACK)
  - `CheckObeliskAttack` — `g_team.c` (MISSIONPACK)
  - All `trap_*` functions — syscall interface to the server engine
