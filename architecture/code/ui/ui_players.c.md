# code/ui/ui_players.c

## File Purpose
Handles 3D player model rendering and animation state management for the Quake III Arena UI. Provides the `UI_DrawPlayer` function used to display animated player characters in menus (character selection, player settings, etc.), along with model/skin/animation loading utilities.

## Core Responsibilities
- Load and register player model parts (legs, torso, head), skins, and weapon models
- Parse `animation.cfg` files to populate animation frame data
- Drive per-frame animation state machines for legs and torso (idle, jump, land, attack, drop/raise weapon)
- Compute hierarchical entity positioning via tag attachment (torso→legs, head→torso, weapon→torso, barrel→weapon)
- Calculate smooth angle transitions (yaw swing, pitch) for the displayed model
- Issue renderer calls to assemble and submit the full player scene each UI frame
- Manage weapon switching sequencing with audio cue

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `playerInfo_t` | struct (typedef, defined in `ui_local.h`) | All state for one UI player display: models, skins, animation frames, weapon, timers |
| `lerpFrame_t` | struct (typedef, defined in `ui_local.h`) | Per-limb interpolation state: old/new frame indices, timing, yaw/pitch swing state |
| `animation_t` | struct (defined in `bg_public.h`) | Single animation descriptor: firstFrame, numFrames, loopFrames, frameLerp, initialLerp |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dp_realtime` | `static int` | file-static | Current timestamp passed into `UI_DrawPlayer`, used throughout frame functions |
| `jumpHeight` | `static float` | file-static | Vertical offset applied to player origin during jump animation |
| `weaponChangeSound` | `sfxHandle_t` | global (extern in `ui_local.h`) | Sound played when weapon switches in the UI preview |

## Key Functions / Methods

### UI_PlayerInfo_SetWeapon
- **Signature:** `static void UI_PlayerInfo_SetWeapon(playerInfo_t *pi, weapon_t weaponNum)`
- **Purpose:** Loads weapon model, barrel model, flash model, and sets flash dlight color for the given weapon. Falls back to `WP_MACHINEGUN`, then `WP_NONE` if models are missing.
- **Inputs:** `pi` — player state; `weaponNum` — desired weapon enum
- **Outputs/Return:** void; mutates `pi->weaponModel`, `pi->barrelModel`, `pi->flashModel`, `pi->flashDlightColor`, `pi->realWeapon`
- **Side effects:** Calls `trap_R_RegisterModel`; uses `goto tryagain` for fallback logic
- **Calls:** `trap_R_RegisterModel`, `COM_StripExtension`, `MAKERGB`
- **Notes:** Only `WP_MACHINEGUN`, `WP_GAUNTLET`, `WP_BFG` get barrel models.

### UI_RunLerpFrame
- **Signature:** `static void UI_RunLerpFrame(playerInfo_t *ci, lerpFrame_t *lf, int newAnimation)`
- **Purpose:** Advances a limb's interpolated frame forward in time relative to `dp_realtime`, handling loop/clamp, and computing `backlerp`.
- **Inputs:** `ci` — animation table source; `lf` — mutable lerp state; `newAnimation` — target animation index
- **Outputs/Return:** void; mutates `lf` fields
- **Side effects:** Reads global `dp_realtime`
- **Calls:** `UI_SetLerpFrameAnimation`
- **Notes:** Clamps `frameTime` to prevent runaway drift (±200ms guard).

### UI_PlayerAnimation
- **Signature:** `static void UI_PlayerAnimation(playerInfo_t *pi, int *legsOld, int *legs, float *legsBackLerp, int *torsoOld, int *torso, float *torsoBackLerp)`
- **Purpose:** Decrements animation timers, runs sequencing state machines, then runs lerp frames; outputs frame indices and backlerp values for the renderer.
- **Inputs:** `pi` — player state; output pointers for legs/torso frame data
- **Side effects:** Modifies `pi->legsAnimationTimer`, `pi->torsoAnimationTimer`; may update global `jumpHeight` via `UI_LegsSequencing`
- **Calls:** `UI_LegsSequencing`, `UI_TorsoSequencing`, `UI_RunLerpFrame`

### UI_DrawPlayer
- **Signature:** `void UI_DrawPlayer(float x, float y, float w, float h, playerInfo_t *pi, int time)`
- **Purpose:** Top-level UI entry point; builds a `refdef_t`, assembles up to 6 `refEntity_t` objects (legs, torso, head, gun, barrel, flash), adds lights, and calls `trap_R_RenderScene`.
- **Inputs:** Screen rect `(x,y,w,h)`, player state `pi`, current time in ms
- **Outputs/Return:** void; submits scene to renderer
- **Side effects:** Sets `dp_realtime`; may fire `weaponChangeSound` via `trap_S_StartLocalSound`; calls `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_RenderScene`
- **Calls:** `UI_AdjustFrom640`, `UI_PlayerAngles`, `UI_PlayerAnimation`, `UI_PositionRotatedEntityOnTag`, `UI_PositionEntityOnTag`, `UI_MachinegunSpinAngle`, `UI_PlayerFloatSprite`, `trap_R_*`, `trap_S_StartLocalSound`
- **Notes:** Returns early if any required model handle is 0 or if `w`/`h` == 0 (cache-only path).

### UI_RegisterClientModelname
- **Signature:** `qboolean UI_RegisterClientModelname(playerInfo_t *pi, const char *modelSkinName, const char *headModelSkinName, const char *teamName)`
- **Purpose:** Parses model/skin name strings, registers all three body part models, registers skins (with team and fallback logic), and loads the animation config.
- **Inputs:** `pi` — destination; name strings with optional `/skinname` suffixes
- **Outputs/Return:** `qtrue` on success, `qfalse` on any failure
- **Side effects:** Calls many `trap_R_RegisterModel`, `trap_R_RegisterSkin`; calls `UI_ParseAnimationFile`

### UI_PlayerInfo_SetModel / UI_PlayerInfo_SetInfo
- **Notes:** Public entry points. `SetModel` zeros `pi` and loads model+default weapon. `SetInfo` drives animation/weapon state changes; handles `newModel` first-frame initialization and death animation forcing.

## Control Flow Notes
- Called from UI menus each frame when a player preview is visible.
- `UI_PlayerInfo_SetModel` is called once on model selection; `UI_PlayerInfo_SetInfo` is called each frame or on state change to update pose.
- `UI_DrawPlayer` is the per-frame render call; it internally advances all animation state, so it must be called with a monotonically increasing `time`.

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `ui_shared.h`
- **Defined elsewhere:** `uiInfo` (global UI context, provides `uiDC.frameTime`); `bg_itemlist` (item/weapon definitions from `bg_misc.c`); all `trap_R_*`, `trap_CM_LerpTag`, `trap_S_*`, `trap_FS_*` syscall wrappers; math utilities (`AnglesToAxis`, `MatrixMultiply`, `AngleSubtract`, `AngleMod`, etc.); animation enum constants (`LEGS_JUMP`, `TORSO_ATTACK`, etc.)
