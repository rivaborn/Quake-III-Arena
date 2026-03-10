# code/game/g_missile.c — Enhanced Analysis

## Architectural Role

`g_missile.c` is the projectile physics and impact hub of the Game VM's weapon/combat subsystem. It sits at the intersection of player intent (fire_* factories called from `g_weapon.c`), deterministic physics (trajectory evaluation via `bg_pmove.c`), and damage application (calls to `g_combat.c`). Every frame, `G_RunMissile` advances all in-flight missiles and dispatches impact logic, making it a critical link in the server's simulation pipeline. MISSIONPACK prox mines demonstrate mid-90s state-machine complexity: entire lifecycle (activation → trigger → detonation) encoded in this single file.

## Key Cross-References

### Incoming (who depends on this file)
- **`g_weapon.c:FireWeapon`** — Calls `fire_plasma`, `fire_grenade`, `fire_rocket`, `fire_bfg`, `fire_grapple` at moment of player weapon discharge; also calls `Weapon_HookFree` for grapple cleanup
- **`g_main.c:G_RunThink`** — Dispatches `G_RunMissile` every server frame for all `ET_MISSILE` entities; controls main loop integration
- **Trigger system** — MISSIONPACK `ProximityMine_Trigger` is registered as touch callback for dynamically-spawned trigger volumes
- **Weapon accuracy** — `g_weapon.c:LogAccuracyHit` called to track per-client hit statistics (increments `client->accuracy_hits`)

### Outgoing (what this file depends on)
- **`bg_pmove.c`/`bg_misc.c`** — `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` for deterministic physics replay; hitTime interpolation in `G_BounceMissile` (line 41–42)
- **`g_combat.c`** — `G_Damage` (direct impact damage, line 285–292), `G_RadiusDamage` (splash, lines 74–79 and 337–343), `CanDamage` (LOS check, MISSIONPACK line 129), `G_InvulnerabilityEffect` (sphere deflection, MISSIONPACK lines 265–273)
- **`g_utils.c`** — `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `G_AddEvent`, `G_SoundIndex`
- **`trap_*` syscalls** — `trap_Trace` (collision detection, line 402), `trap_LinkEntity` (spatial indexing, lines 79, 197, 330, etc.)
- **Global state** — `level.time`, `level.previousTime` (timing), `g_entities[]` (entity lookups), `g_proxMineTimeout`, `g_gametype` (gamer rules)

## Design Patterns & Rationale

**Trajectory Replay via `BG_EvaluateTrajectory`**: Missiles encode movement as a kinematic trajectory tuple `(origin, velocity, gravity_accel, start_time, type)` and evaluate it deterministically at any point in time. This allows lag-compensated hitTime interpolation (line 41–42) without keeping per-frame position history. Elegant but requires careful time management.

**Factory Pattern (`fire_*` functions)**: Each weapon has its own spawn function (plasma, rocket, grenade, BFG, grapple, and MISSIONPACK nail + prox). This avoids type-dispatch conditionals in a single factory and allows per-weapon tuning (speed: plasma/BFG 2000, rocket 900, grenade 700, grapple 800 ups). All use `level.time - MISSILE_PRESTEP_TIME` to ensure visible first-frame movement.

**Entity State Machine via Think/Nextthink**: Instead of per-frame conditional checks, missiles use `ent->think` function pointers and `ent->nextthink` timestamps (idiomatic to Quake). Prox mines exemplify this: `ProximityMine_Activate` → armed state, `ProximityMine_Trigger` callback → detonation, `ProximityMine_ExplodeOnPlayer` → stuck-on-player countdown. No scripting; all in C.

**Event-Driven Network Updates via `G_AddEvent`**: Impact events (`EV_MISSILE_HIT`, `EV_MISSILE_MISS`, `EV_MISSILE_MISS_METAL`, `EV_GRENADE_BOUNCE`, etc.) are emitted as entity events, decoupling client-side effects from game state changes. cgame then interprets these to spawn effects, sounds, decals.

**Grapple Hook's Dual-Entity Pattern** (lines 310–334): The hook spawns a separate visual-only entity (`nent`, `ET_GENERAL`) at impact and converts the original missile (`ent`) to `ET_GRAPPLE` for continuous attachment. The visual entity is freed after its event is played; the grapple continues pulling. This separation avoids state conflicts between visual representation and pull logic.

**Proximity Mine Trigger Volume** (lines 160–197): Rather than checking distance every frame on the mine entity, a separate trigger volume is spawned with the mine as parent. This leverages the collision system's AABB query efficiency and keeps damage checks (`CanDamage`) localized.

## Data Flow Through This File

```
Player fires weapon (g_weapon.c:FireWeapon)
  ↓
fire_X() [e.g., fire_rocket]
  Creates gentity_t with ET_MISSILE, trajectory (TR_LINEAR + velocity)
  Sets parent, ownerNum, damage, splashDamage, methodOfDeath
  Returns missile entity
  ↓
[Every server frame]
G_RunMissile(ent) [called from g_main.c:G_RunThink]
  Evaluates trajectory at level.time → current position
  Determines passent (owner to ignore, or ENTITYNUM_NONE for prox post-exit)
  Traces movement from previous origin to current origin
  If collision (tr.fraction < 1):
    → G_MissileImpact(ent, &tr)
        ├─ Bounce check (EF_BOUNCE / EF_BOUNCE_HALF)
        │   → G_BounceMissile: reflect velocity, damp, possibly stop
        ├─ Invulnerability deflection (MISSIONPACK only)
        │   → G_InvulnerabilityEffect: deflect off sphere
        ├─ Direct damage (if other->takedamage)
        │   → G_Damage: apply impact damage, track accuracy_hits
        ├─ Special: Prox mine sticking (MISSIONPACK + WP_PROX_LAUNCHER)
        │   → ProximityMine_Player: attach to player, set EF_TICKING
        ├─ Special: Grapple hook (classname=="hook")
        │   → Spawn visual nent, convert ent to ET_GRAPPLE
        │   → Weapon_HookThink per-frame attachment
        └─ Default: Splash damage
            → G_RadiusDamage: area damage, track accuracy_hits
        Then: G_AddEvent (HIT/MISS/MISS_METAL/BOUNCE), set freeAfterEvent
  Else (no collision):
    Continue next frame
    ↓
[Timed explosion or event trigger]
G_ExplodeMissile(ent) [timeout or prox trigger]
  Evaluates final position, sets ET_GENERAL
  Emits EV_MISSILE_MISS event
  Applies splash damage if configured
  Sets freeAfterEvent
```

## Learning Notes

**Idiomatic to Quake engine era (1999–2005)**:
- No scene graph, component system, or prefab hierarchy; everything is a flat `gentity_t` array
- Physics is pure trajectory math: no ragdoll, no complex collision shapes — just AABB sweeps
- Network efficiency prioritized: entity events separate from state updates; delta compression elsewhere
- Accuracy tracking is per-shot, incremented only on successful hits (not just near-misses), feeding into player skill statistics

**MISSIONPACK complexity**: Proximity mines showcase the game evolving mid-development. The entire mine lifecycle (spawn → arm after 2s → trigger sphere → stick to player → detonate with timeout) is hardcoded in a handful of functions, no script system. This was likely retrofitted for Team Arena.

**Deterministic Lag Compensation**: `G_BounceMissile` interpolates hitTime (line 41–42) by using `trace->fraction` to replay the exact moment of impact within the frame. This avoids frame-boundary artifacts and is critical for netplay fairness.

**Grapple as Pseudo-Tether**: The grapple hook is not a full ragdoll or constraint solver; it's a simple pull-toward-hook target with `Weapon_HookThink` positioning the player each frame. This is fast and predictable but lacks modern physics fidelity.

**Accuracy Semantics**: `accuracy_hits` is incremented only on projectile *impact*, not firing. This differs from hitscan weapons where hits are instant. The prox mine also credits the original owner if it kills someone later, making team damage tracking complex.

## Potential Issues

1. **Implicit State in `ent->count`** (MISSIONPACK): The prox mine uses `ent->count` as a flag for "has left owner bbox" (line 405), but this is never clearly documented. A named bitfield or explicit state enum would be clearer.

2. **`G_MissileImpact` Complexity**: The function has grown to handle bounces, invulnerability deflection, direct damage, prox sticking, grapple attachment, and splash damage—nearly 150 lines with nested conditionals. Extracting helper functions (e.g., `G_ApplyDirectDamage`, `G_ApplySplashDamage`) would improve readability.

3. **Prox Mine Team Check Only in MISSIONPACK** (line 126): The proximity mine's team awareness is conditional. Base Q3A has no equivalent, meaning prox mines would trigger on teammates in the original game. This could be a balance oversight or intentional.

4. **Entity Leak Risk in Grapple**: If `Weapon_HookThink` or the grapple sequence is interrupted (e.g., hook owner quits), the visual `nent` is marked `freeAfterEvent` and should be freed by the event system. If that system is ever changed, the leak could resurface.

5. **Sky Surface Handling Upstream**: The file checks `SURF_NOIMPACT` in `G_RunMissile` (line 411), not `G_MissileImpact`. This is correct but fragile: if `G_MissileImpact` is ever called directly (e.g., from a different code path), sky hits won't be handled. A safety check inside `G_MissileImpact` would be more defensive.
