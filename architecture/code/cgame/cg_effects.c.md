# code/cgame/cg_effects.c

## File Purpose
Generates client-side visual effects as local entities, primarily in response to game events such as weapon impacts, player deaths, teleportation, and special item activations. All effects are purely cosmetic and client-local, not networked.

## Core Responsibilities
- Spawn bubble trail local entities for underwater projectiles
- Create smoke puff / blood trail local entities with configurable color, fade, and velocity
- Generate explosion local entities (sprite and model variants)
- Spawn player gib fragments with randomized gravity trajectories
- Handle teleport, score plum, and MissionPack-exclusive effects (Kamikaze, Obelisk, Invulnerability)
- Emit positional sounds for pain and impact events (Obelisk, Invulnerability)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `localEntity_t` | struct (typedef) | Client-side entity with lifetime, trajectory, color, and render data |
| `refEntity_t` | struct | Renderer-facing entity embedded in `localEntity_t` |
| `leType_t` | enum | Classifies local entity behavior (explosion, fragment, fade, etc.) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `seed` (in `CG_SmokePuff`) | `static int` | file-static local | Persistent random seed for sprite rotation |
| `lastPos` (in `CG_ScorePlum`) | `static vec3_t` | file-static local | Tracks last score plum origin to prevent overlap |

## Key Functions / Methods

### CG_BubbleTrail
- Signature: `void CG_BubbleTrail( vec3_t start, vec3_t end, float spacing )`
- Purpose: Spawns a chain of bubble sprites along a line segment for underwater projectile trails.
- Inputs: World-space start/end points, spacing distance between bubbles.
- Outputs/Return: None; allocates local entities as side effect.
- Side effects: Allocates `localEntity_t` objects; reads `cg.time`, `cgs.media.waterBubbleShader`, `cg_noProjectileTrail`.
- Calls: `CG_AllocLocalEntity`, `VectorCopy`, `VectorSubtract`, `VectorNormalize`, `VectorMA`, `VectorScale`, `VectorAdd`, `random`, `crandom`.
- Notes: Random initial offset prevents all trails from being in-phase. Early-out if `cg_noProjectileTrail` is set.

### CG_SmokePuff
- Signature: `localEntity_t *CG_SmokePuff( const vec3_t p, const vec3_t vel, float radius, float r, float g, float b, float a, float duration, int startTime, int fadeInTime, int leFlags, qhandle_t hShader )`
- Purpose: Creates a single smoke or blood puff local entity with full color/fade control.
- Inputs: Position, velocity, radius, RGBA color, duration, timing, flags, shader.
- Outputs/Return: Pointer to the allocated `localEntity_t`.
- Side effects: Allocates a `localEntity_t`; falls back to `smokePuffRageProShader` on `GLHW_RAGEPRO` hardware.
- Calls: `CG_AllocLocalEntity`, `Q_random`, `VectorCopy`.
- Notes: Caller is responsible for not holding the pointer beyond the entity's lifetime. `fadeInTime` controls whether `lifeRate` is computed from fade-in or start.

### CG_MakeExplosion
- Signature: `localEntity_t *CG_MakeExplosion( vec3_t origin, vec3_t dir, qhandle_t hModel, qhandle_t shader, int msec, qboolean isSprite )`
- Purpose: Core factory for explosion local entities; supports both sprite and 3D model variants.
- Inputs: Origin, direction (for axis rotation), model/shader handles, lifetime, sprite flag.
- Outputs/Return: Pointer to allocated `localEntity_t`.
- Side effects: Allocates a `localEntity_t`; calls `CG_Error` if `msec <= 0`.
- Calls: `CG_AllocLocalEntity`, `CG_Error`, `VectorScale`, `VectorAdd`, `VectorCopy`, `AxisClear`, `RotateAroundDirection`.
- Notes: Applies a random time offset (0–63 ms) to desynchronize multiple simultaneous explosions. Sprite explosions offset origin along `dir` by 16 units.

### CG_GibPlayer
- Signature: `void CG_GibPlayer( vec3_t playerOrigin )`
- Purpose: Spawns a burst of gib fragment local entities from a player death origin.
- Inputs: Player world-space origin.
- Outputs/Return: None.
- Side effects: Allocates multiple `localEntity_t` via `CG_LaunchGib`; early-outs if `cg_blood` is 0. Skull/brain always spawn; remaining gibs require `cg_gibs`.
- Calls: `CG_LaunchGib`, `VectorCopy`, `crandom`, `rand`.

### CG_LaunchGib
- Signature: `void CG_LaunchGib( vec3_t origin, vec3_t velocity, qhandle_t hModel )`
- Purpose: Allocates a single gib fragment with gravity trajectory, bounce, blood marks, and bounce sound.
- Inputs: Origin, velocity, model handle.
- Outputs/Return: None.
- Side effects: Allocates a `localEntity_t`; sets `LE_FRAGMENT` type with `TR_GRAVITY`.
- Calls: `CG_AllocLocalEntity`, `VectorCopy`, `AxisCopy`.

### CG_Bleed
- Signature: `void CG_Bleed( vec3_t origin, int entityNum )`
- Purpose: Spawns a blood explosion sprite at impact point; hides it from first-person view for the local player.
- Inputs: World origin, entity number of the hit entity.
- Outputs/Return: None.
- Side effects: Allocates a `localEntity_t`; sets `RF_THIRD_PERSON` renderfx if `entityNum` is the local client.
- Calls: `CG_AllocLocalEntity`, `VectorCopy`, `rand`.
- Notes: Early-out if `cg_blood` is 0.

### CG_SpawnEffect
- Signature: `void CG_SpawnEffect( vec3_t org )`
- Purpose: Creates a teleport visual effect model local entity at the given origin.
- Inputs: World origin.
- Outputs/Return: None.
- Side effects: Allocates a `localEntity_t`; origin Z is adjusted by ±24/+16 depending on `MISSIONPACK` define.
- Calls: `CG_AllocLocalEntity`, `VectorCopy`, `AxisClear`.

### Notes on Minor Functions
- `CG_ScorePlum`: Spawns a floating score display only for the local scoring client; uses `lastPos` to prevent z-overlap.
- `CG_LightningBoltBeam` (MISSIONPACK): Short-lived `LE_SHOWREFENTITY` beam between two points using `RT_LIGHTNING`.
- `CG_KamikazeEffect`, `CG_ObeliskExplode`, `CG_InvulnerabilityImpact`, `CG_InvulnerabilityJuiced` (MISSIONPACK): Specialized model-based effects with sounds.
- `CG_ObeliskPain` (MISSIONPACK): Pure audio effect, randomly selects one of three hit sounds.
- `CG_LaunchExplode` / `CG_BigExplode`: Variant of gib system using `cgs.media.smoke2` with lower bounce (0.1) and brass sound type.

## Control Flow Notes
This file is called during the **frame update** path, not during init or shutdown. Functions here are invoked by `cg_events.c` (event processing) and `cg_weapons.c` (weapon effects) during each rendered frame. The allocated `localEntity_t` objects are subsequently processed and rendered by `cg_localents.c:CG_AddLocalEntities()` each frame until their `endTime` expires.

## External Dependencies
- `cg_local.h`: All cgame types (`localEntity_t`, `cg_t`, `cgs_t`, `leType_t`, etc.)
- `cg.time`: Current client render time (global `cg_t`)
- `cgs.media.*`: Preloaded shader/model handles (global `cgs_t`)
- `cgs.glconfig.hardwareType`: GPU capability check for RagePro fallback
- **Defined elsewhere**: `CG_AllocLocalEntity` (`cg_localents.c`), `CG_MakeExplosion` (this file, called by `CG_ObeliskExplode`), `trap_S_StartSound` (syscall layer), `AxisClear`, `RotateAroundDirection`, `VectorNormalize`, `AnglesToAxis` (math library), `axisDefault` (global defined in renderer/shared code)
