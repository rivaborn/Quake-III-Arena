# code/q3_ui/ui_players.c

## File Purpose
Implements the animated 3D player model preview rendering used in the Q3 UI (e.g., player selection screens). It manages model/skin/weapon loading, animation state machines for legs and torso, and submits all player-related render entities to the renderer each frame.

## Core Responsibilities
- Load and register player model parts (legs, torso, head), skins, and weapon models
- Parse `animation.cfg` files to populate the `animation_t` array
- Drive per-frame animation state machines for legs and torso (sequencing, blending, jump arcs)
- Compute hierarchical bone/tag placement for torso, head, gun, barrel, and muzzle flash entities
- Submit the full multi-part player entity (+ lights, sprite) to the renderer via `trap_R_*` syscalls
- Handle weapon-switch transitions, muzzle flash timing, and barrel spin for machine-gun-style weapons

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `playerInfo_t` | struct (typedef, defined in `ui_local.h`) | All state for one preview player: models, skins, animation frames, weapon, timing |
| `lerpFrame_t` | struct (typedef, defined in `ui_local.h`) | Per-limb animation interpolation state (frame, oldframe, backlerp, yaw/pitch swing) |
| `animation_t` | struct (from `bg_public.h`) | Describes one animation clip: firstFrame, numFrames, loopFrames, frameLerp, initialLerp |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `dp_realtime` | `static int` | file-static | Current time snapshot set at the start of each `UI_DrawPlayer` call |
| `jumpHeight` | `static float` | file-static | Current vertical offset applied to the player origin during jump animation |

## Key Functions / Methods

### UI_DrawPlayer
- **Signature:** `void UI_DrawPlayer( float x, float y, float w, float h, playerInfo_t *pi, int time )`
- **Purpose:** Main entry point — renders the complete animated player model into a 2D screen rect.
- **Inputs:** Screen rect (x,y,w,h), player state `pi`, current time in ms.
- **Outputs/Return:** void; submits render commands via syscalls.
- **Side effects:** Sets `dp_realtime`; triggers pending weapon swap and sound; calls `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_RenderScene`.
- **Calls:** `UI_AdjustFrom640`, `UI_PlayerAngles`, `UI_PlayerAnimation`, `UI_PositionRotatedEntityOnTag`, `UI_PositionEntityOnTag`, `UI_MachinegunSpinAngle`, `UI_PlayerFloatSprite`, `trap_R_*`, `trap_S_StartLocalSound`.
- **Notes:** Returns early if any model or animation is missing. Adds two accent point lights at fixed world offsets.

### UI_RegisterClientModelname
- **Signature:** `qboolean UI_RegisterClientModelname( playerInfo_t *pi, const char *modelSkinName )`
- **Purpose:** Loads the three `.md3` model parts and skins for `"modelname/skinname"`, then parses the animation config.
- **Inputs:** Destination `pi`, combined `"model/skin"` string.
- **Outputs/Return:** `qtrue` on full success; `qfalse` on any load failure.
- **Side effects:** Registers renderer resources via `trap_R_RegisterModel/Skin`; populates `pi->legsModel`, `torsoModel`, `headModel`, `legsSkin`, `torsoSkin`, `headSkin`, `animations[]`.
- **Calls:** `UI_RegisterClientSkin`, `UI_ParseAnimationFile`, `trap_R_RegisterModel`.

### UI_PlayerInfo_SetModel
- **Signature:** `void UI_PlayerInfo_SetModel( playerInfo_t *pi, const char *model )`
- **Purpose:** Full reset of `pi` and load of a new model, setting default weapon to machinegun.
- **Side effects:** Zeroes `*pi`, calls `UI_RegisterClientModelname`, `UI_PlayerInfo_SetWeapon`.

### UI_PlayerInfo_SetInfo
- **Signature:** `void UI_PlayerInfo_SetInfo( playerInfo_t *pi, int legsAnim, int torsoAnim, vec3_t viewAngles, vec3_t moveAngles, weapon_t weaponNumber, qboolean chat )`
- **Purpose:** Updates per-frame display state: view/move angles, pending animations, weapon transitions, chat icon, death handling.
- **Side effects:** May call `UI_ForceLegsAnim`, `UI_ForceTorsoAnim`, `UI_PlayerInfo_SetWeapon`; sets `muzzleFlashTime`.

### UI_RunLerpFrame
- **Signature:** `static void UI_RunLerpFrame( playerInfo_t *ci, lerpFrame_t *lf, int newAnimation )`
- **Purpose:** Advances a lerp frame using `dp_realtime`, handling animation switching, looping, and backlerp calculation.
- **Side effects:** Mutates `lf` fields (`frame`, `oldFrame`, `frameTime`, `backlerp`).

### UI_PlayerAngles
- **Signature:** `static void UI_PlayerAngles( playerInfo_t *pi, vec3_t legs[3], vec3_t torso[3], vec3_t head[3] )`
- **Purpose:** Computes per-body-part rotation axes from view/move angles, applying swing damping and movement direction adjustment.
- **Calls:** `UI_SwingAngles`, `UI_MovedirAdjustment`, `AnglesToAxis`, `AnglesSubtract`.

### UI_ParseAnimationFile
- **Signature:** `static qboolean UI_ParseAnimationFile( const char *filename, animation_t *animations )`
- **Purpose:** Reads and parses an `animation.cfg` text file into the `animations[]` array; adjusts leg frame offsets.
- **Side effects:** File I/O via `trap_FS_*`; writes to `animations[0..MAX_ANIMATIONS-1]`.

### Notes
- `UI_PlayerInfo_SetWeapon` iterates `bg_itemlist` to look up the weapon item and loads barrel/flash sub-models; uses `goto tryagain` fallback to machinegun.
- `UI_PositionEntityOnTag` / `UI_PositionRotatedEntityOnTag` call `trap_CM_LerpTag` to attach child entities to parent bone tags.
- `UI_SwingAngles` implements a non-linear angle-spring for yaw/pitch following.
- `UI_MachinegunSpinAngle` drives barrel roll/pitch spin with coast-down when not firing.

## Control Flow Notes
- Called from UI menu draw callbacks each frame.
- `UI_PlayerInfo_SetModel` is called once on model selection; `UI_PlayerInfo_SetInfo` is called every frame to push new animation/angle intent; `UI_DrawPlayer` is called every frame to advance state and render.
- There is no separate init/shutdown; all state lives in caller-owned `playerInfo_t`.

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`
- **Defined elsewhere:** `bg_itemlist` (game item table), `uis` (global `uiStatic_t`), `weaponChangeSound`, all `trap_*` syscall wrappers, math utilities (`AnglesToAxis`, `MatrixMultiply`, `VectorMA`, etc.), animation constants (`LEGS_JUMP`, `TORSO_ATTACK`, `ANIM_TOGGLEBIT`, `MAX_ANIMATIONS`, etc.)
