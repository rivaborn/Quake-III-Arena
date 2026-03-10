# code/cgame/cg_weapons.c — Enhanced Analysis

## Architectural Role

`cg_weapons.c` sits at the intersection of **cgame's input/event processing** and **scene rendering pipeline**. It translates server-delivered weapon events and snapshots into renderer submissions (models, lights) and audio output (fire sounds, impact feedback), while spawning client-side ephemeral effects (trails, brass, explosions) that reduce network bandwidth by omitting server sync. It is the primary bridge between the game VM's entity/event stream and the two "peripheral" subsystems: renderer and sound.

## Key Cross-References

### Incoming (who depends on this file)

- **`cg_view.c`** → `CG_AddViewWeapon`, `CG_AddPlayerWeapon` (called during per-frame scene construction)
- **`cg_events.c`** → `CG_FireWeapon`, `CG_MissileHitWall`, `CG_MissileHitPlayer`, `CG_ShotgunFire` (entity event dispatch callbacks)
- **`cg_draw.c`** → `CG_DrawWeaponSelect` (HUD drawing)
- **`cg_main.c`** → `CG_RegisterWeapon`, `CG_RegisterItemVisuals` (level load media precaching)

### Outgoing (what this file depends on)

- **Renderer**: `trap_R_RegisterModel`, `trap_R_RegisterShader`, `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_ModelBounds`
- **Sound**: `trap_S_RegisterSound`, `trap_S_StartSound`, `trap_S_AddLoopingSound`
- **Physics**: `BG_EvaluateTrajectory`, `CG_PointContents`, `CG_Trace` (via trap_CM)
- **Local entities**: `CG_AllocLocalEntity`, `CG_SmokePuff`, `CG_MakeExplosion`, `CG_ImpactMark`, `CG_BubbleTrail`, `CG_ParticleExplosion`, `CG_Bleed`
- **Entity attachment**: `CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag` (from `cg_ents.c`)
- **Global state**: `cg`, `cgs` (master cgame state), `cg_entities[]`, `cg_weapons[]`, `cg_items[]`

## Design Patterns & Rationale

### 1. **Lazy Registration with Idempotency**
Each weapon registers its media on first use via a `registered` flag. This defers I/O cost until gameplay requires the weapon, and the guard prevents redundant re-registration.

### 2. **Effect Composition from Local Entities**
Complex effects are built by spawning multiple short-lived `localEntity_t` instances (e.g., rail trail = 1 core beam + N rotating sprite rings). This leverages the fixed 512-entity pool and amortizes per-entity rendering overhead.

### 3. **Client-Side Determinism for Bandwidth Saving**
Shotgun pellet decals/sounds are replayed client-side using `es->eventParm` as RNG seed, exactly matching server physics. This eliminates per-pellet network messages while keeping visuals consistent. **Tradeoff**: architecture couples client behavior to server RNG implementation.

### 4. **Cvar-Driven Behavior Trees**
Weapon effects obey runtime cvars (`cg_brassTime`, `cg_noProjectileTrail`, `cg_oldRail`, `cg_trueLightning`) to enable quick artist iteration and diagnostic toggling. No hard-coded quality tiers.

### 5. **Weapon-Specific Vtable Pointers**
Trail rendering is dispatched via `weaponInfo_t.trailFunc` (function pointer), allowing per-weapon trail behavior (rocket smoke vs. nail puff vs. rail helical rings) without type-switching.

## Data Flow Through This File

```
WEAPON REGISTRATION (map load):
  cg_main.c::CG_RegisterItems
    → CG_RegisterItemVisuals → CG_RegisterWeapon
    → trap_R_RegisterModel(barrel/flash variants)
    → trap_S_RegisterSound(fire sounds)
    → CG_RegisterItemVisuals (secondary world models for holdables/armor)

FIRST-PERSON RENDERING (per-frame):
  cg_view.c::CG_RenderView
    → CG_AddViewWeapon (hand pose + FOV adjustment)
    → CG_AddPlayerWeapon (attach weapon + barrel + muzzle flash to torso)
    → trap_R_AddRefEntityToScene (weapon + flash ref entities)
    → trap_R_AddLightToScene (muzzle dynamic light)

FIRE EVENT (network):
  Server entity event → cg_events.c::CG_EntityEvent
    → CG_FireWeapon
    → (1) Set muzzle flash time (controls LE_MUZZLE_FLASH lifetime)
    → (2) trap_S_StartSound (fire audio)
    → (3) Eject brass → CG_MachineGunEjectBrass/etc. (LE_FRAGMENT × N)
    → (4) For hitscan (LG): CG_LightningBolt → renderer trace + impact flare
    → (5) For projectiles: CG_SpawnRailTrail/etc. (per-frame trail spawning)

PROJECTILE TRAIL (per-frame):
  cg_localents.c::CG_AddLocalEntities
    → For each active entity, evaluate trail position
    → wi->trailFunc(ent, wi) → CG_RocketTrail/CG_RailTrail/CG_PlasmaTrail
    → CG_SmokePuff (LE_SCALE_FADE) spawned at intervals
    → Trails accumulate in local entity pool, retired after lifetime

IMPACT EVENT (network):
  Server entity event → cg_events.c::CG_EntityEvent
    → CG_MissileHitWall(weapon, origin, direction, soundType)
    → (1) CG_MakeExplosion (LE_EXPLOSION_CHUNK × 6)
    → (2) CG_ImpactMark (permanent decal at origin)
    → (3) trap_S_StartSound (metal/flesh impact variant)
    → (4) CG_ParticleExplosion (dust cloud for rockets)
```

## Learning Notes

### For Quake III Engine Students

1. **Bandwidth-Conscious Architecture**: Pre-computed weapon media (models, shaders, sounds) cached at registration time avoids redundant allocations. Deterministic client-side effects (trails, decals) eliminate per-frame network overhead.

2. **Shader Composition**: Effects like rail trails use custom shaders (`RT_RAIL_CORE`, `RT_SPRITE`) that don't exist in the BSP. The renderer's `RT_*` type enum allows dynamic geometry.

3. **Temporal Interpolation**: All projectile effects (trails, smoke) are spawned at discrete intervals (`step = 50`ms) using `BG_EvaluateTrajectory` to find historical positions, then interpolated per-frame by the renderer's lerp code.

4. **Weapon Attachment Hierarchy**: The first-person weapon attaches to a *predicted* hand/torso tag; the third-person weapon attaches to a *server-derived* torso entity, creating latency-visible weapon sway.

### Modern Engines Do Differently

- **Dynamic asset loading**: Modern engines stream shader/model assets on-demand, not bulk-precached at level load.
- **Data-driven weapon configs**: Separate `.json`/`.yaml` files define trail radius, lifetime, colors—not compile-time constants or cvar magic numbers.
- **Effect graph retained rendering**: Trail and decal lifetime are authored in a retained effect graph (Unreal Blueprint, Unity VFX Graph), not imperative spawning.
- **GPU particle systems**: The 512-entity local entity pool would be replaced by GPU compute for thousands of particles with no CPU overhead.

### Idiomatic Q3 Patterns

1. **`trailTime` for frame delta**: Projectiles track `trailTime` to emit effects only if enough time passed, avoiding duplicate frames.
2. **Crandom blend for variance**: Velocity and lifetime include `crandom()` ([-1, 1]) to reduce repetitiveness without per-effect randomness seeds.
3. **Water-aware scaling**: Brass and projectiles apply `waterScale = 0.1f` in water to reduce density, a cheap physics hack.

## Potential Issues

### 1. **Hardcoded Model Path Conventions**
Functions derive flash/barrel/hand variants by stripping `.md3` and appending suffix (line ~1150 area). If a weapon MD3 structure breaks this convention, registration silently fails with a null model handle, creating invisible flash effects at runtime. **Risk**: asset pipeline breakage is subtle.

### 2. **Magic Numbers Throughout**
Offset vectors (`offset[0]=8`, `offset[1]=-4`), lifetime constants (`100+50*crandom()`), and trail spacing (`SPACING=5`) are scattered. Tweaking physics requires code edits and recompilation. **Rationale**: Q3 prioritized runtime simplicity over data-driven flexibility (2005 era).

### 3. **No Registration Validation**
`CG_RegisterWeapon` does not check if `trap_R_RegisterModel` succeeds; a missing model leaves `re->hModel = NULL`, causing silent render-time failures. Modern engines would log and fallback.

### 4. **Shotgun Seed Dependency**
`CG_ShotgunFire` relies on `es->eventParm` as RNG seed to replay client decals identically to server. Any change to server event parm layout breaks client-side decal placement. **Fragility**: cross-module API coupling.

### 5. **Legacy Cvar Branches**
`cg_oldRail` (line ~242) suggests old rail trail rendering was retained as a compatibility fallback. Maintaining two code paths increases cognitive load.

### 6. **Muzzle Flash Lifetime Hardcoded**
`MUZZLE_FLASH_TIME` (undefined in excerpt, likely ~150ms) is baked into the fire event handler. Changing muzzle flash duration requires code edit, not a cvar.
