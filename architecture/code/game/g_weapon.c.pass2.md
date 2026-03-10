# code/game/g_weapon.c — Enhanced Analysis

## Architectural Role

This file implements the **server-authoritative weapon firing pipeline** for the Game VM subsystem. It bridges player input events (from `g_active.c`) to two divergent paths: immediate hitscan damage (traces + `G_Damage` calls) for melee/hitscan weapons, and entity spawning (via `g_missile.c`) for projectiles. The file executes entirely within the game VM sandbox, making no direct engine calls except through `trap_*` syscalls, and is fundamental to the server's authority over all damage and hit detection—a critical property in multiplayer gameplay.

## Key Cross-References

### Incoming (who depends on this file)
- **`g_active.c:ClientThink()`** — Calls `FireWeapon()` once per firing event and `CheckGauntletAttack()` each frame gauntlet is held
- **`g_missile.c`** — Not a call dependency; rather, `g_weapon.c` spawns projectiles via `fire_rocket()`, `fire_grenade()`, `fire_plasma()`, `fire_bfg()`, `fire_grapple()`, `fire_nail()`, `fire_prox()` (all defined in `g_missile.c`)
- **Game VM callers** (implicitly) — Any code that fires weapons (bots via `be_ai_weap.c` trap calls, players via client input)

### Outgoing (what this file depends on)
- **`g_combat.c`** — `G_Damage()`, `G_InvulnerabilityEffect()` (MISSIONPACK bouncing), `G_RadiusDamage()`
- **`g_missile.c`** — All projectile fire functions (`fire_rocket`, `fire_grenade`, `fire_plasma`, `fire_bfg`, `fire_grapple`, `fire_nail`, `fire_prox`)
- **`g_local.h` globals** — `g_entities[]`, `level`, `g_quadfactor` (cvar), `g_gametype` (cvar), `g_knockback` (MISSIONPACK)
- **Engine via `trap_*` syscalls** — `trap_Trace()`, `trap_LinkEntity()`, `trap_UnlinkEntity()`, `trap_EntitiesInBox()` (collision queries), `trap_BotLib*()` (bot integration)
- **`cgame` VM (indirectly)** — Spawns temp entities (`G_TempEntity()`) that the cgame consumes as `EV_*` events for client-side effects

## Design Patterns & Rationale

### 1. **Fire-Event Amortization via File-Statics**
The file-static vectors (`forward`, `right`, `up`, `muzzle`, `s_quadFactor`) are computed once in `FireWeapon()` and remain valid throughout all nested weapon-firing sub-calls. This pattern avoids recomputing muzzle position or quad multipliers across multiple hitscan traces or projectile launches within a single firing event. The pattern is **not thread-safe** but is safe within the VM's single-threaded frame loop.

### 2. **Server Authoritarianism via Hitscan Traces**
All hitscan weapons (`Bullet_Fire`, `weapon_railgun_fire`, `Weapon_LightningFire`, `CheckGauntletAttack`) execute `trap_Trace()` server-side to compute hit detection and damage, preventing client-side cheat mods from claiming "lucky" hits. Clients predict their own damage locally (in cgame VM via `CG_ShotgunPattern`) but the server's trace is authoritative; if the client's prediction diverges, the server snapshot corrects it.

### 3. **Deterministic Prediction via Seeded PRNG**
`ShotgunPattern()` uses a seeded `Q_crandom()` call to generate spread angles, with the seed provided as an event parameter. This ensures the client-side `CG_ShotgunPattern()` (cgame VM) computes identical pellet trajectories as the server, allowing client-side visual feedback to match server damage without retransmitting each pellet's result.

### 4. **Penetration via Spatial Unlink**
The railgun unlinks hit entities from the world (via `trap_UnlinkEntity()`) after hitting them, allowing the trace to continue through them. After processing all hits, entities are relinked (via `trap_LinkEntity()`). This avoids O(N²) collision checks and is cheaper than clipping-plane math for multi-target penetration.

### 5. **MISSIONPACK Invulnerability Sphere as Bidirectional Mechanic**
The invulnerability sphere (`traceEnt->client->invulnerabilityTime > level.time`) can **both defend and attack**. When hit, it calls `G_InvulnerabilityEffect()` which bounces the incoming hitscan ray (via `G_BounceProjectile()`) off a computed impact point, allowing the bounce to hit other targets or the shooter themselves. This is a late-game power fantasy: "I'm invincible AND I bounce your shots back at you."

### 6. **Damage Multipliers as Runtime Powerup State**
Quad Damage and Doubler are **checked at fire time**, not baked into weapon properties. `FireWeapon()` queries `ent->client->ps.powerups[PW_QUAD]` and `persistantPowerup->item->giTag` (MISSIONPACK) to compute `s_quadFactor`. This decouples weapon definitions from powerup mechanics and allows powerups to affect all weapons uniformly without per-weapon updates.

## Data Flow Through This File

```
Player fires weapon (via g_active.c ClientThink)
    ↓
FireWeapon(ent)
    - Compute forward/right/up vectors from player viewangles
    - Compute muzzle point (player origin + viewheight + 14 units forward)
    - Set s_quadFactor from powerups
    - Dispatch to per-weapon function
    ↓
    ┌─────────────────────────┬─────────────────────────┐
    ↓ (Hitscan weapons)       ↓ (Projectile weapons)   
    CheckGauntletAttack(),    BFG_Fire(),
    Bullet_Fire(),            Weapon_RocketLauncher_Fire(),
    weapon_railgun_fire(),    Weapon_Plasmagun_Fire(),
    Weapon_LightningFire()    weapon_grenadelauncher_fire(),
                              Weapon_HookFire()
    ↓                         ↓
trap_Trace() to hit point    fire_*() spawns entity_t
    ↓                         ↓
G_Damage(hitEnt, ...)        Entity simulated each frame
                             by g_missile.c think funcs
    ↓                         ↓
    └─────────────────────────┴─────────────────────────┐
                               ↓
                    G_TempEntity() (EV_* event)
                               ↓
                    cgame VM receives event
                    → sound, effect, HUD feedback
```

## Learning Notes

### Game Engine Concepts
- **Hitscan vs Projectile Duality**: Modern engines (Unreal, Unity) often unify these, but Q3 keeps them separate—hitscan is instant server authority, projectiles are continuous simulation. This teaches the tension between frame-perfect detection vs perceived smoothness.
- **Prediction Reconciliation**: The cgame's local prediction (shotgun pellets) uses the same seed as the server, avoiding desync messages for casual players while remaining cheater-proof for competitive play.
- **Powerups as Orthogonal Multipliers**: Rather than baking Quad into each weapon, the engine checks it once per fire event. This is simpler than inheriting from a "QuadWeapon" class and avoids explosion of weapon variants.

### Idiomatic to Q3 Era / Early 2000s Design
- **Fixed Pool Allocation**: Temp entities, local entities, projectiles—all preallocated arrays with reuse. Modern engines use dynamic allocation and GC, but Q3's pool-based approach guarantees frame-time predictability.
- **Compile-Time Variants via `#ifdef`**: MISSIONPACK features are conditionally compiled, not runtime-selectable. This is crude by modern standards (feature flags, plugins) but kept binary size low on dial-up.
- **VM Sandbox for Cheater Resistance**: By running game logic in a bytecode VM, the server can audit all damage claims. A native DLL would allow clients to modify game state directly; a VM prevents that.

### Connections to Modern Engine Concepts
- **ECS (Entity Component System)**: This file treats weapons as passive data (damage value, spread) + active fire functions. A modern ECS might define a `WeaponComponent` and a `FireSystem`. Here, there's no weapon entity; the entity firing has the weapon.
- **Networking**: The `EV_*` temp entities are the RPC equivalent—lightweight events telling the client "something happened" without transmitting full entity state.

## Potential Issues

1. **`CalcMuzzlePointOrigin` Vestigial Parameter** — The `origin` parameter is passed but never used; identical body to `CalcMuzzlePoint`. Dead code or placeholder for future refactoring.

2. **Truncation in `SnapVectorTowards`** — Rounds endpos to integers for bandwidth; could introduce visible misalignment if the snapped point diverges significantly from the hit point. Not verified against client prediction in this file (cgame's burden).

3. **Railgun Unlink/Relink Correctness** — If `G_Damage()` or `LogAccuracyHit()` attempts to access the hit entity's spatial links before relinking, undefined behavior. Mitigation relies on game loop invariant (damage doesn't immediately trigger spatial changes).

4. **Kamikaze Damage vs Map Boundaries** — `G_StartKamikaze` damage expands over 1 second; if the player leaves the map or is teleported during detonation, the expanding damage sphere may behave unexpectedly. Not explicitly handled.

5. **Quad/Doubler Cvar Not Validated** — `g_quadfactor.value` is read directly; no range check. A malicious or misconfigured server could set it to an extreme value, affecting all damage globally.
