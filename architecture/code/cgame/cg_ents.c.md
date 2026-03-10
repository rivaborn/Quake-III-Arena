# code/cgame/cg_ents.c

## File Purpose
Presents server-transmitted snapshot entities to the renderer and sound system every frame. It resolves interpolated/extrapolated positions for all `centity_t` objects and dispatches per-type rendering logic (players, missiles, movers, items, etc.).

## Core Responsibilities
- Compute per-frame lerp/extrapolated origins and angles for all packet entities via `CG_CalcEntityLerpPositions`
- Apply continuous per-entity effects (looping sounds, constant lights) via `CG_EntityEffects`
- Dispatch entity-type-specific rendering through `CG_AddCEntity` (switch on `eType`)
- Attach child render entities to parent model tags (`CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag`)
- Adjust entity positions when riding movers (`CG_AdjustPositionForMover`)
- Drive the auto-rotation state (`cg.autoAngles/autoAxis`) used by all world items
- Submit the local predicted player entity in addition to server-sent entities

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `centity_t` | struct | Client-side entity; holds current/next `entityState_t`, lerp origin/angles, and misc timing state |
| `refEntity_t` | struct | Renderer submission record; filled per-entity and passed to `trap_R_AddRefEntityToScene` |
| `weaponInfo_t` | struct | Per-weapon media/trail/sound references; consulted for missiles and grapple |
| `itemInfo_t` | struct | Per-item model/icon handles; consulted for `CG_Item` |
| `orientation_t` | struct | Tag orientation returned by `trap_R_LerpTag`; used in tag-attachment helpers |

## Global / File-Static State
None defined in this file. All shared state is accessed through the externally-defined globals `cg`, `cgs`, `cg_entities`, `cg_weapons`, and `cg_items`.

## Key Functions / Methods

### CG_PositionEntityOnTag
- **Signature:** `void CG_PositionEntityOnTag(refEntity_t *entity, const refEntity_t *parent, qhandle_t parentModel, char *tagName)`
- **Purpose:** Positions a child render entity at a named bone/tag of a parent model, composing their axes.
- **Inputs:** Child entity to move, parent entity, parent model handle, tag name string.
- **Outputs/Return:** Modifies `entity->origin`, `entity->axis`, `entity->backlerp` in place.
- **Side effects:** Calls `trap_R_LerpTag`.
- **Calls:** `trap_R_LerpTag`, `VectorCopy`, `VectorMA`, `MatrixMultiply`.
- **Notes:** Casts away `const` on parent to satisfy `MatrixMultiply`; the child's pre-existing axis is **replaced** (not premultiplied with its own rotation).

### CG_PositionRotatedEntityOnTag
- **Signature:** `void CG_PositionRotatedEntityOnTag(refEntity_t *entity, const refEntity_t *parent, qhandle_t parentModel, char *tagName)`
- **Purpose:** Same as above but premultiplies the child's existing axis into the result, preserving its own rotation.
- **Inputs/Outputs:** Same pattern as `CG_PositionEntityOnTag`.
- **Calls:** `trap_R_LerpTag`, `MatrixMultiply` (twice).

### CG_SetEntitySoundPosition
- **Signature:** `void CG_SetEntitySoundPosition(centity_t *cent)`
- **Purpose:** Updates the 3-D sound origin for an entity; BSP brush models use their geometric midpoint.
- **Side effects:** Calls `trap_S_UpdateEntityPosition`.

### CG_EntityEffects
- **Signature:** `static void CG_EntityEffects(centity_t *cent)`
- **Purpose:** Applies continuous per-entity audio and lighting: looping sounds and constant-light glows.
- **Side effects:** `trap_S_AddLoopingSound`, `trap_S_AddRealLoopingSound`, `trap_R_AddLightToScene`, calls `CG_SetEntitySoundPosition`.
- **Notes:** `ET_SPEAKER` entities use real (non-attenuated) looping sound; others use standard looping sound.

### CG_Item
- **Signature:** `static void CG_Item(centity_t *cent)`
- **Purpose:** Renders a world item with bobbing, autorotation, scale-up on respawn, and optional secondary ring/sphere model.
- **Side effects:** Mutates `cent->lerpOrigin` (bob offset, weapon midpoint correction); calls `trap_R_AddRefEntityToScene` one or more times.
- **Notes:** `cg_simpleItems` causes sprite rendering instead. Weapons are scaled 1.5×; under `MISSIONPACK`, kamikaze holdable is scaled 2×.

### CG_Missile
- **Signature:** `static void CG_Missile(centity_t *cent)`
- **Purpose:** Renders a projectile: invokes trail function, adds dynamic light, adds looping velocity-aware sound, submits render entity.
- **Side effects:** `weapon->missileTrailFunc(cent, weapon)`, `trap_R_AddLightToScene`, `trap_S_AddLoopingSound`, `trap_R_AddRefEntityToScene`, `CG_AddRefEntityWithPowerups`.
- **Notes:** Plasma gun uses `RT_SPRITE`; other missiles rotate around their travel axis via `RotateAroundDirection`.

### CG_AdjustPositionForMover
- **Signature:** `void CG_AdjustPositionForMover(const vec3_t in, int moverNum, int fromTime, int toTime, vec3_t out)`
- **Purpose:** Offsets a position by the delta movement of a mover entity between two server times; used for entities riding platforms.
- **Calls:** `BG_EvaluateTrajectory` (4 times).
- **Notes:** Angular correction is not implemented (marked FIXME). Early-outs for invalid mover indices or non-`ET_MOVER` entities.

### CG_CalcEntityLerpPositions
- **Signature:** `static void CG_CalcEntityLerpPositions(centity_t *cent)`
- **Purpose:** Computes `cent->lerpOrigin` and `cent->lerpAngles` for the current frame via interpolation or trajectory evaluation.
- **Calls:** `CG_InterpolateEntityPosition`, `BG_EvaluateTrajectory`, `CG_AdjustPositionForMover`.
- **Notes:** Client entities with `cg_smoothClients` off are forced to `TR_INTERPOLATE`. The predicted player entity skips mover adjustment.

### CG_AddCEntity
- **Signature:** `static void CG_AddCEntity(centity_t *cent)`
- **Purpose:** Top-level per-entity dispatch: computes lerp positions, applies effects, then calls the appropriate type handler.
- **Calls:** `CG_CalcEntityLerpPositions`, `CG_EntityEffects`, then one of `CG_General/CG_Player/CG_Item/CG_Missile/CG_Mover/CG_Beam/CG_Portal/CG_Speaker/CG_Grapple/CG_TeamBase`.
- **Notes:** Entities with `eType >= ET_EVENTS` are skipped (handled elsewhere).

### CG_AddPacketEntities
- **Signature:** `void CG_AddPacketEntities(void)`
- **Purpose:** Frame entry point; computes `frameInterpolation`, seeds auto-rotation axes, submits the predicted player entity, then iterates all snapshot entities calling `CG_AddCEntity`.
- **Side effects:** Writes `cg.frameInterpolation`, `cg.autoAngles/autoAxis/autoAnglesFast/autoAxisFast`; calls `BG_PlayerStateToEntityState`; calls `CG_AddCEntity` for every entity in `cg.snap`.

## Control Flow Notes
`CG_AddPacketEntities` is called once per rendered frame (from `CG_DrawActiveFrame` in `cg_view.c`). It is the root of all entity presentation. Interpolation state set here feeds into `CG_CalcEntityLerpPositions` for every entity. Tag-attachment helpers are called from weapon/player rendering code in `cg_weapons.c` and `cg_players.c`.

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `tr_types.h`, `cg_public.h`
- **Defined elsewhere:** `CG_Player` (`cg_players.c`), `CG_AddRefEntityWithPowerups` (`cg_players.c`), `CG_GrappleTrail` (`cg_weapons.c`), `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta`, `BG_PlayerStateToEntityState` (`bg_misc.c`/`bg_pmove.c`)
- Renderer traps: `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_LerpTag`
- Sound traps: `trap_S_UpdateEntityPosition`, `trap_S_AddLoopingSound`, `trap_S_AddRealLoopingSound`, `trap_S_StartSound`
- Math utilities: `VectorCopy/MA/Add/Subtract/Scale/Clear/Normalize2`, `AnglesToAxis`, `MatrixMultiply`, `AxisCopy/Clear`, `RotateAroundDirection`, `PerpendicularVector`, `CrossProduct`, `ByteToDir`, `LerpAngle`
