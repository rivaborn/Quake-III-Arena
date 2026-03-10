# code/cgame/cg_view.c

## File Purpose
Sets up all 3D rendering parameters (view origin, view angles, FOV, viewport rect) each frame and issues the final render call. It is the central per-frame orchestration point for the cgame's visual output.

## Core Responsibilities
- Compute viewport rectangle based on `cg_viewsize` cvar
- Offset first-person or third-person view with bobbing, damage kick, step smoothing, duck smoothing, and land bounce
- Calculate FOV with zoom interpolation and underwater warp
- Build and submit the `refdef_t` to the renderer via `CG_DrawActive`
- Add all scene entities (packet entities, marks, particles, local entities, view weapon, test model)
- Manage a circular sound buffer for announcer/sequential sounds
- Emit powerup-expiry warning sounds
- Provide developer model-testing commands (`testmodel`, `testgun`, frame/skin cycling)

## Key Types / Data Structures
None defined in this file; relies on types from `cg_local.h`.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `mins` / `maxs` (inside `CG_OffsetThirdPersonView`) | `static vec3_t` | static local | Collision box half-extents for third-person camera trace |

## Key Functions / Methods

### CG_TestModel_f
- **Signature:** `void CG_TestModel_f(void)`
- **Purpose:** Console command — spawns a test entity 100 units in front of the view for visual model inspection.
- **Inputs:** Console args: `<modelpath> [backlerp]`
- **Outputs/Return:** void; populates `cg.testModelEntity` and `cg.testModelName`
- **Side effects:** Calls `trap_R_RegisterModel`; writes `cg.testGun = qfalse`
- **Calls:** `trap_Argc`, `CG_Argv`, `trap_R_RegisterModel`, `VectorMA`, `AnglesToAxis`, `CG_Printf`
- **Notes:** Model must be a full path from basedir.

### CG_TestGun_f
- **Signature:** `void CG_TestGun_f(void)`
- **Purpose:** Variant of `CG_TestModel_f` that attaches the model to the view weapon slot.
- **Side effects:** Sets `cg.testGun = qtrue`; adds `RF_MINLIGHT | RF_DEPTHHACK | RF_FIRST_PERSON` render flags.
- **Calls:** `CG_TestModel_f`

### CG_AddTestModel (static)
- **Signature:** `static void CG_AddTestModel(void)`
- **Purpose:** Each frame, re-registers and submits the test model entity to the scene.
- **Side effects:** Calls `trap_R_AddRefEntityToScene`
- **Notes:** For gun mode, origin is derived from `cg.refdef.vieworg` plus `cg_gun_x/y/z` offsets.

### CG_CalcVrect (static)
- **Signature:** `static void CG_CalcVrect(void)`
- **Purpose:** Computes `cg.refdef.{x,y,width,height}` from `cg_viewsize` and GL resolution; forces full-screen during intermission.
- **Side effects:** May call `trap_Cvar_Set` to clamp `cg_viewsize` to [30,100].

### CG_OffsetThirdPersonView (static)
- **Signature:** `static void CG_OffsetThirdPersonView(void)`
- **Purpose:** Repositions the camera behind/beside the player for third-person mode; traces against geometry to prevent clipping.
- **Inputs:** Reads `cg.refdef`, `cg.predictedPlayerState`, `cg_thirdPersonRange`, `cg_thirdPersonAngle`, `cg_cameraMode`
- **Side effects:** Modifies `cg.refdef.vieworg` and `cg.refdefViewAngles`; calls `CG_Trace`
- **Notes:** Dead players have yaw locked to `STAT_DEAD_YAW`; focus distance is `512` units.

### CG_OffsetFirstPersonView (static)
- **Signature:** `static void CG_OffsetFirstPersonView(void)`
- **Purpose:** Applies all first-person view perturbations: weapon kick, damage kick, velocity pitch/roll, head-bob, duck/step/land smoothing.
- **Side effects:** Modifies `cg.refdef.vieworg` and `cg.refdefViewAngles` in place.
- **Calls:** `CG_StepOffset`, `VectorAdd`, `DotProduct`
- **Notes:** Returns early during `PM_INTERMISSION`; dead players get fixed roll/pitch angles.

### CG_CalcFov (static)
- **Signature:** `static int CG_CalcFov(void)`
- **Purpose:** Computes `fov_x`/`fov_y` with zoom interpolation and sinusoidal underwater warp; sets `cg.zoomSensitivity`.
- **Outputs/Return:** `int inwater` — nonzero if view is in water/slime/lava
- **Side effects:** Writes `cg.refdef.fov_x`, `cg.refdef.fov_y`, `cg.zoomSensitivity`
- **Calls:** `CG_PointContents`, `sin`, `atan2`, `tan`

### CG_CalcViewValues (static)
- **Signature:** `static int CG_CalcViewValues(void)`
- **Purpose:** Master function that populates `cg.refdef` — calls `CG_CalcVrect`, handles intermission, computes bobbing state, applies error decay, dispatches first/third person offset, converts angles to axis, sets hyperspace flags.
- **Outputs/Return:** `int inwater` from `CG_CalcFov`
- **Side effects:** Clears `cg.refdef`; writes many `cg` fields (`bobcycle`, `bobfracsin`, `xyspeed`, etc.)

### CG_DamageBlendBlob (static)
- **Signature:** `static void CG_DamageBlendBlob(void)`
- **Purpose:** Adds a fading blood-sprite `refEntity_t` in front of the camera to represent received damage.
- **Side effects:** Calls `trap_R_AddRefEntityToScene`; skips on GLHW_RAGEPRO hardware.

### CG_AddBufferedSound
- **Signature:** `void CG_AddBufferedSound(sfxHandle_t sfx)`
- **Purpose:** Enqueues a sound into a circular buffer for sequential announcer playback.
- **Side effects:** Writes `cg.soundBuffer`, advances `cg.soundBufferIn`.

### CG_DrawActiveFrame
- **Signature:** `void CG_DrawActiveFrame(int serverTime, stereoFrame_t stereoView, qboolean demoPlayback)`
- **Purpose:** Main per-frame entry point called by the client engine. Drives the complete scene build and render pipeline.
- **Inputs:** `serverTime` — authoritative time; `stereoView` — left/right/mono; `demoPlayback` flag
- **Side effects:** Updates `cg.time`, processes snapshots, runs prediction, builds refdef, populates scene, calls `CG_DrawActive`, updates lagometer and timescale fade.
- **Calls:** `CG_UpdateCvars`, `CG_ProcessSnapshots`, `CG_PredictPlayerState`, `CG_CalcViewValues`, `CG_DamageBlendBlob`, `CG_AddPacketEntities`, `CG_AddMarks`, `CG_AddParticles`, `CG_AddLocalEntities`, `CG_AddViewWeapon`, `CG_PlayBufferedSounds`, `CG_PlayBufferedVoiceChats`, `CG_AddTestModel`, `CG_PowerupTimerSounds`, `trap_S_Respatialize`, `CG_AddLagometerFrameInfo`, `CG_DrawActive`, `trap_R_ClearScene`, `trap_S_ClearLoopingSounds`, `trap_SetUserCmdValue`
- **Notes:** Skips full scene build and returns early (showing loading screen) if `cg.infoScreenText` is set or no valid snapshot exists. Lagometer / timescale updates are skipped for `STEREO_RIGHT` to avoid double-counting.

## Control Flow Notes
`CG_DrawActiveFrame` is the cgame's frame entry point, called once per rendered frame by the client (`cl_cgame.c`). It follows the sequence: time update → cvar sync → snapshot processing → prediction → view setup → scene population → audio spatialization → final draw. It is the top of the cgame render call graph.

## External Dependencies
- `cg_local.h` — all cgame types, `cg`, `cgs`, cvars, trap declarations
- `trap_R_*` — renderer scene API (defined in engine)
- `trap_S_*` — sound API (defined in engine)
- `CG_DrawActive` — defined in `cg_draw.c`
- `CG_PredictPlayerState`, `CG_Trace`, `CG_PointContents` — defined in `cg_predict.c`
- `CG_AddPacketEntities` — defined in `cg_ents.c`
- `CG_AddViewWeapon` — defined in `cg_weapons.c`
- `CG_PlayBufferedVoiceChats` — defined in `cg_servercmds.c`
- `AnglesToAxis`, `VectorMA`, `DotProduct`, `AngleVectors` — math utilities from `q_math.c`
