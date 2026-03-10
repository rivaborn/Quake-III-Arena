# code/game/g_combat.c — Enhanced Analysis

## Architectural Role

`g_combat.c` is the **authoritative damage pipeline hub** within the Game VM subsystem (`code/game/`). All weapon impacts from `g_missile.c` and `g_weapon.c`, environmental hazards from `g_trigger.c` and `g_active.c`, and mover crushers from `g_mover.c` converge on `G_Damage` as the sole bottleneck before any entity health change occurs. This centralization means all cross-cutting concerns — team protection, armor, invulnerability, handicap scaling, debug logging — are enforced in exactly one place. The file sits at the junction between the physics/weapon layer (which computes what was hit) and the entity lifecycle layer (`g_client.c`, `g_items.c`, `g_team.c`) which handles the consequences of dying.

The `player_die` function doubles as the **client entity's `->die` vtable slot**, installed during `ClientSpawn` in `g_client.c`; this is the primary integration point between the combat system and the client lifecycle managed by the server.

## Key Cross-References

### Incoming (who depends on this file)

| Caller | What it calls | Why |
|--------|--------------|-----|
| `g_missile.c` | `G_Damage`, `G_RadiusDamage` | Rocket/grenade/plasma explosion and direct impact |
| `g_weapon.c` | `G_Damage`, `LogAccuracyHit` (inverse — weapon defines it, combat calls it) | Hitscan weapons (railgun, lightning, machinegun) |
| `g_trigger.c` | `G_Damage` | `trigger_hurt` volumes, teleporter lethal zones |
| `g_active.c` | `G_Damage` | Lava/slime/fall damage, drowning |
| `g_mover.c` | `G_Damage` | Door/platform crush |
| `g_client.c` | `player_die` (installed as `ent->die`), `TossClientItems` | Client entity death callback and respawn setup |
| `g_team.c` | `AddScore`, `G_Damage` (via `Team_FragBonuses` calling back) | CTF/team scoring side effects |

`modNames[]` is consumed only within this file (for `G_LogPrintf`); it is not exported.

### Outgoing (what this file depends on)

| Subsystem | Functions/globals accessed | Notes |
|-----------|--------------------------|-------|
| `g_team.c` | `Team_FragBonuses`, `Team_ReturnFlag`, `Team_CheckHurtCarrier`, `OnSameTeam`, `CheckObeliskAttack` | Team rule enforcement during damage and death |
| `g_weapon.c` | `Weapon_HookFree`, `LogAccuracyHit`, `G_StartKamikaze` (MISSIONPACK) | Hook cleanup and hit-accuracy bookkeeping |
| `g_items.c` | `Drop_Item`, `LaunchItem`, `BG_FindItemForWeapon`, `BG_FindItemForPowerup`, `BG_FindItem` | Item drops on death |
| `g_cmds.c` | `Cmd_Score_f` | Force-sends updated scoreboard to all clients after a kill |
| `g_utils.c` | `G_TempEntity`, `G_Spawn`, `G_Find`, `G_FreeEntity`, `G_AddEvent`, `G_EntitiesFree` | Entity allocation and event broadcasting |
| `g_main.c` | `level` global, `g_entities[]`, `CalculateRanks` | Level state, entity array, rank recalculation |
| `qcommon/vm` syscalls | `trap_Trace`, `trap_EntitiesInBox`, `trap_LinkEntity` | Engine-boundary calls for collision and entity queries |
| cvars | `g_knockback`, `g_blood`, `g_friendlyFire`, `g_gametype`, `g_debugDamage`, `g_cubeTimeout` | Runtime tuning of combat behavior |

## Design Patterns & Rationale

**Entity vtable (function pointers for `->die` / `->pain`)**: Rather than a monolithic type switch, each entity class installs its own death handler. `player_die` and `body_die` are the two installed here. This is a manual vtable — a 1999 solution to the same problem modern engines solve with component dispatch or ECS. The tradeoff is simplicity at the cost of no runtime type safety.

**Single-entry damage funnel**: Every source of damage calls `G_Damage` rather than modifying `ent->health` directly. This makes it trivially safe to add new global conditions (handicap, warmup immunity, godmode) in one place. The architectural cost is that all callers must be disciplined about routing through this function — there is no enforcement mechanism beyond convention.

**Temp entity broadcast for visual feedback**: Score plums (`EV_SCOREPLUM`), obituaries (`EV_OBITUARY`), and gibs (`EV_GIB_PLAYER`) are all communicated to clients as `gentity_t` temp entities rather than through a separate notification channel. This reuses the existing snapshot/entity-delta system for event propagation, avoiding a separate messaging bus — a Q3-era economy of mechanism.

**`static int` round-robin for animation**: The cycling death animation counter in `player_die` uses a file-static variable modified each call. This is intentionally non-deterministic across kills, which is acceptable for aesthetic variety. It would be a bug if two players died in the same frame and both needed independent animation state — which cannot happen in single-threaded game code.

## Data Flow Through This File

```
Weapon/Trigger/Environment
         │
         ▼
    G_Damage(targ, inflictor, attacker, dir, point, damage, dflags, mod)
         │
         ├─ [Invulnerability?] → G_InvulnerabilityEffect → return bounce dir
         ├─ [Armor?] → CheckArmor → reduce damage, drain armor stat
         ├─ [Team fire?] → OnSameTeam → scale/block damage
         ├─ Apply knockback → targ->client->ps velocity delta
         ├─ Accumulate damage_* fields (for pain-direction HUD)
         │
         ├─ [health > 0] → targ->pain() callback
         └─ [health ≤ 0] → targ->die() → player_die()
                                │
                                ├─ G_LogPrintf (kill log)
                                ├─ G_TempEntity(EV_OBITUARY) → broadcast to all clients
                                ├─ AddScore → CalculateRanks → Cmd_Score_f
                                ├─ Team_FragBonuses / Team_ReturnFlag
                                ├─ TossClientItems → Drop_Item (world entities)
                                ├─ LookAtKiller → STAT_DEAD_YAW
                                └─ GibEntity or death animation → trap_LinkEntity

G_RadiusDamage(origin, attacker, damage, radius, ignore, mod)
         │
         ├─ trap_EntitiesInBox (spatial query → entity list)
         ├─ For each entity: CanDamage (up to 5 trap_Trace calls)
         └─ G_Damage with falloff = damage * (1 - dist/radius)
```

The `damage_*` accumulation fields written in `G_Damage` are read by `g_active.c`'s `ClientEndFrame` to generate the per-frame pain direction indicator in the HUD — a deferred cross-frame data dependency invisible from this file alone.

## Learning Notes

- **VM isolation is pervasive**: Every engine service is accessed through a `trap_*` wrapper. `g_combat.c` cannot call `CM_Trace` directly; it must call `trap_Trace`. This enforces the QVM security boundary — the game module is sandboxed and can be loaded as bytecode. Modern engines simply link everything together or use explicit permission systems.

- **`bg_*` shared layer significance**: `BG_FindItemForWeapon` and friends are compiled into *both* game and cgame. This is Q3's solution to prediction consistency — item and weapon metadata must be identical client and server side. Modern engines handle this with data-driven asset definitions.

- **The `->die` vtable predates ECS by 20 years**: `g_combat.c`'s use of `targ->die()` and `targ->pain()` function pointers is a hand-rolled component behavior pattern. The entity struct in `g_local.h` is essentially a fat component blob with optional fields, predating the explicit component separation that ECS formalizes.

- **`CheckAlmostCapture` / `CheckAlmostScored` as proximity reward signals**: These XOR-toggle the `PLAYEREVENT_HOLYSHIT` flag, which cgame translates to a sound stinger. This is an early example of contextual game-feel feedback driven by spatial proximity checks at the moment of a kill — a design pattern now common in modern shooters as "clutch" or "ace" notifications.

- **`CanDamage` multi-trace**: Firing 5 traces (center + 4 box corners in XY) for splash damage LOS is a practical approximation. It can produce inconsistent results at geometry edges where some traces clear and others don't, but avoids the cost of true volumetric occlusion. Modern engines often use similar multi-point sampling.

## Potential Issues

- **`static int i` animation cycle**: Non-reentrant by construction; safe only because the game runs single-threaded. If the game VM were ever moved to a multi-threaded model this would be a data race.
- **`G_Find` loop in `CheckAlmostCapture`**: Called from `player_die` every death; iterates the full entity list via linked-list search. At high player counts with frequent deaths, this is O(n) per kill. Not a practical issue at Q3's 64-entity client cap, but worth noting.
- **`modNames[]` bounds check is asymmetric**: The check `meansOfDeath >= sizeof(modNames)/sizeof(modNames[0])` correctly catches out-of-range values but only for logging — the actual `mod` value is passed unmodified to `G_Damage` callers and ultimately to cgame for client-side obituary text. A mismatch between server and client `modNames` tables would produce garbled obituaries silently.
