# code/cgame/cg_weapons.c

## File Purpose
Client-side weapon visualization module for Quake III Arena. Handles all weapon-related rendering, effects, and input, including view weapon display, projectile trails, muzzle flashes, impact effects, shell ejection, and weapon selection UI.

## Core Responsibilities
- Register and cache weapon/item media (models, shaders, sounds) at level load
- Render the first-person view weapon with bobbing, FOV offset, and animation mapping
- Render world-space weapon models attached to player entities (with powerup overlays)
- Emit per-weapon trail effects (rocket smoke, rail rings, plasma sparks, grapple beam)
- Spawn muzzle flash, dynamic light, and brass ejection local entities on fire events
- Resolve hitscan impact effects (explosions, marks, sounds) for all weapon types
- Simulate shotgun pellet spread client-side (matching server seed) for decals/sounds
- Manage weapon cycling (next/prev/direct select) and on-screen weapon selection HUD

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `weaponInfo_t` | struct | Per-weapon media cache: models, sounds, trail function pointer, flash color |
| `itemInfo_t` | struct | Per-item media cache: world models, icon shader |
| `centity_t` | struct | Client entity with lerped origin/angles, trail time, player entity state |
| `localEntity_t` | struct | Short-lived effect entity (brass, smoke, rail ring, explosion sprite) |
| `impactSound_t` | enum | Impact sound variant: DEFAULT, METAL, FLESH |
| `leType_t` | enum | Local entity behavior type (LE_FRAGMENT, LE_FADE_RGB, LE_SCALE_FADE, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cg_weapons[MAX_WEAPONS]` | `weaponInfo_t[]` | global (extern) | Registered weapon info table indexed by WP_* |
| `cg_items[MAX_ITEMS]` | `itemInfo_t[]` | global (extern) | Registered item info table indexed by item num |
| `cg` | `cg_t` | global (extern) | Master cgame state (time, player state, refdef) |
| `cgs` | `cgs_t` | global (extern) | Static cgame state (media handles, client infos) |
| `cg_entities[MAX_GENTITIES]` | `centity_t[]` | global (extern) | All client-side entities |

## Key Functions / Methods

### CG_RegisterWeapon
- Signature: `void CG_RegisterWeapon( int weaponNum )`
- Purpose: Load all media for a weapon on first use; idempotent via `registered` flag.
- Inputs: `weaponNum` — WP_* enum value
- Outputs/Return: void; populates `cg_weapons[weaponNum]`
- Side effects: Calls `trap_R_RegisterModel/Shader`, `trap_S_RegisterSound`; writes shared `cgs.media` fields (e.g. `lightningShader`, `railRingsShader`)
- Calls: `CG_RegisterItemVisuals`, `trap_R_ModelBounds`, `COM_StripExtension`
- Notes: Weapon 0 is a no-op. Derives flash/barrel/hand model paths by stripping `.md3` extension and appending suffix.

### CG_RegisterItemVisuals
- Signature: `void CG_RegisterItemVisuals( int itemNum )`
- Purpose: Load world model and icon for a game item; calls `CG_RegisterWeapon` for weapon items.
- Inputs: `itemNum` — index into `bg_itemlist`
- Outputs/Return: void; populates `cg_items[itemNum]`
- Side effects: Allocates renderer handles.
- Notes: Bounds-checks `itemNum`; secondary world model loaded for powerups/armor/health/holdables.

### CG_AddPlayerWeapon
- Signature: `void CG_AddPlayerWeapon( refEntity_t *parent, playerState_t *ps, centity_t *cent, int team )`
- Purpose: Attach and submit weapon, barrel, and flash ref entities to the render scene for a player entity.
- Inputs: `parent` — torso ref entity; `ps` — non-NULL for local player; `cent` — owning entity; `team`
- Outputs/Return: void
- Side effects: Calls `trap_R_AddRefEntityToScene`, `trap_S_AddLoopingSound`, `trap_R_AddLightToScene`; triggers `CG_LightningBolt`, `CG_SpawnRailTrail`
- Notes: Railgun barrel receives custom RGBA based on refire fraction. Muzzle flash suppressed after `MUZZLE_FLASH_TIME` ms unless continuous-fire weapon.

### CG_AddViewWeapon
- Signature: `void CG_AddViewWeapon( playerState_t *ps )`
- Purpose: Compute first-person hand position and call `CG_AddPlayerWeapon` for the local player.
- Inputs: `ps` — predicted player state
- Outputs/Return: void
- Side effects: Reads `cg_fov`, `cg_gun_x/y/z` cvars; maps torso animation frames to weapon frames via `CG_MapTorsoToWeaponFrame`
- Notes: Skipped for spectators, intermission, third-person, hidden gun (`cg_drawGun`), or test gun mode.

### CG_FireWeapon
- Signature: `void CG_FireWeapon( centity_t *cent )`
- Purpose: Handle `EV_FIRE_WEAPON` event — set muzzle flash time, play fire sound, eject brass.
- Inputs: `cent` — firing entity
- Side effects: Sets `cent->muzzleFlashTime`; calls `trap_S_StartSound`, `weap->ejectBrassFunc`
- Notes: Lightning gun skips repeat events while already firing.

### CG_MissileHitWall
- Signature: `void CG_MissileHitWall( int weapon, int clientNum, vec3_t origin, vec3_t dir, impactSound_t soundType )`
- Purpose: Spawn explosion model, impact mark decal, and sound for a weapon impact on geometry.
- Inputs: weapon type, shooter client (for rail color), hit position/normal, surface sound type
- Side effects: `CG_MakeExplosion`, `CG_ImpactMark`, `trap_S_StartSound`, `CG_ParticleExplosion` (rockets)
- Notes: Railgun impact is colorized with `cgs.clientinfo[clientNum].color1/2`.

### CG_RailTrail
- Signature: `void CG_RailTrail( clientInfo_t *ci, vec3_t start, vec3_t end )`
- Purpose: Spawn the rail core beam and, unless `cg_oldRail`, a helical ring of sprite local entities.
- Inputs: `ci` — shooter client info for color; start/end world positions
- Side effects: Allocates multiple `localEntity_t` (LE_FADE_RGB core + LE_MOVE_SCALE_FADE rings)
- Notes: Ring spacing, radius, and rotation defined by `SPACING=5`, `RADIUS=4`, `ROTATION=1` macros.

### CG_ShotgunFire
- Signature: `void CG_ShotgunFire( entityState_t *es )`
- Purpose: Client-side replay of shotgun pellet traces from entity state to produce decals and sounds.
- Calls: `CG_ShotgunPattern` → `CG_ShotgunPellet` → `CG_MissileHitWall/Player`
- Notes: Uses `es->eventParm` as random seed to replicate server spread exactly.

### CG_LightningBolt
- Signature: `static void CG_LightningBolt( centity_t *cent, vec3_t origin )`
- Purpose: Trace and render the LG beam from muzzle to impact; add impact flare model if it hit.
- Notes: Supports `cg_trueLightning` cvar to blend between predicted and lerped view angles.

## Control Flow Notes
- **Init**: `CG_RegisterWeapon` / `CG_RegisterItemVisuals` called lazily on first encounter or at map load via `CG_LoadingItem`.
- **Per-frame**: `CG_AddViewWeapon` → `CG_AddPlayerWeapon` is called from `cg_view.c` during scene construction. `CG_DrawWeaponSelect` is called during 2D HUD drawing.
- **Events**: `CG_FireWeapon`, `CG_MissileHitWall`, `CG_MissileHitPlayer`, `CG_ShotgunFire` are called from `cg_events.c` in response to entity events replayed from snapshots.

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `BG_EvaluateTrajectory` — defined in `bg_misc.c`
- `CG_AllocLocalEntity`, `CG_SmokePuff`, `CG_MakeExplosion`, `CG_BubbleTrail`, `CG_Bleed`, `CG_ImpactMark`, `CG_ParticleExplosion` — defined in other cgame modules
- `CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag` — `cg_ents.c`
- `trap_R_*`, `trap_S_*`, `trap_CM_*` — VM syscall stubs (`cg_syscalls.c`)
- `axisDefault`, `vec3_origin` — defined in `q_math.c` / `q_shared.c`
