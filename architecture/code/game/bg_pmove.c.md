# code/game/bg_pmove.c

## File Purpose
Implements the core player movement (pmove) system for Quake III Arena, shared between the server and client game modules. Takes a `pmove_t` (containing a `playerState_t` and `usercmd_t`) as input and produces an updated `playerState_t` as output. Designed for deterministic client-side prediction.

## Core Responsibilities
- Simulates all player movement modes: walking, air, water, fly, noclip, grapple, dead, spectator
- Applies friction and acceleration per-medium (ground, water, flight, spectator)
- Detects and handles ground contact, slope clamping, and the "all solid" edge case
- Manages jump, crouch/duck, water level, and water jump logic
- Drives weapon state transitions (raising, dropping, firing, ammo consumption)
- Drives legs and torso animation state machines via toggle-bit animation indices
- Generates predictable player events (footsteps, splashes, fall damage, weapon fire, etc.)
- Chops long frames into sub-steps via `Pmove` to prevent framerate-dependent behavior

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `pmove_t` | struct (typedef) | Top-level pmove context: playerstate ptr, command, trace callbacks, results | 
| `playerState_t` | struct (typedef) | Full player state mutated by pmove; transmitted over net |
| `usercmd_t` | struct (typedef) | Client input command (movement axes, buttons, weapon, serverTime) |
| `pml_t` | struct (typedef) | Per-frame pmove locals: forward/right/up axes, frametime, ground trace, previous state |
| `trace_t` | struct (typedef) | Collision trace result used for ground and duck checks |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pm` | `pmove_t *` | global | Current pmove context pointer; set at entry of `PmoveSingle` |
| `pml` | `pml_t` | global | Per-frame locals; zeroed at start of each `PmoveSingle` call |
| `pm_stopspeed` | `float` | global | Minimum speed threshold for friction control (100.0) |
| `pm_duckScale` | `float` | global | Speed scale while ducked (0.25) |
| `pm_swimScale` | `float` | global | Speed scale while swimming (0.50) |
| `pm_wadeScale` | `float` | global | Speed scale while wading (0.70) |
| `pm_accelerate` | `float` | global | Ground acceleration rate (10.0) |
| `pm_airaccelerate` | `float` | global | Air acceleration rate (1.0) |
| `pm_wateraccelerate` | `float` | global | Water acceleration rate (4.0) |
| `pm_flyaccelerate` | `float` | global | Flight acceleration rate (8.0) |
| `pm_friction` | `float` | global | Ground friction (6.0) |
| `pm_waterfriction` | `float` | global | Water friction (1.0) |
| `pm_flightfriction` | `float` | global | Flight powerup friction (3.0) |
| `pm_spectatorfriction` | `float` | global | Spectator friction (5.0) |
| `c_pmove` | `int` | global | Frame counter for debug journaling |

## Key Functions / Methods

### Pmove
- **Signature:** `void Pmove(pmove_t *pmove)`
- **Purpose:** Public entry point. Subdivides large time deltas into ≤66ms (or `pmove_msec`) sub-steps to ensure framerate-independent movement.
- **Inputs:** `pmove` — fully initialized pmove context with current `playerState_t` and `usercmd_t`.
- **Outputs/Return:** Mutates `pmove->ps` in place.
- **Side effects:** Calls `PmoveSingle` one or more times; maintains `pmove->ps->pmove_framecount`.
- **Calls:** `PmoveSingle`
- **Notes:** If `pmove_fixed` is set, uses `pmove_msec` as the fixed step size. Jump-held upmove is re-injected between sub-steps.

### PmoveSingle
- **Signature:** `void PmoveSingle(pmove_t *pmove)`
- **Purpose:** Executes one complete movement frame: sets up locals, selects movement mode, runs weapon/animation/event logic.
- **Inputs:** `pmove` — single-step pmove context.
- **Outputs/Return:** Mutates `pm->ps`.
- **Side effects:** Sets global `pm`, zeros `pml`, calls `trap_SnapVector` on velocity at end.
- **Calls:** `PM_UpdateViewAngles`, `PM_CheckDuck`, `PM_GroundTrace`, `PM_SetWaterLevel`, `PM_DeadMove`, `PM_DropTimers`, `PM_FlyMove`, `PM_GrappleMove`, `PM_WaterJumpMove`, `PM_WaterMove`, `PM_WalkMove`, `PM_AirMove`, `PM_Animate`, `PM_Weapon`, `PM_TorsoAnimation`, `PM_Footsteps`, `PM_WaterEvents`
- **Notes:** Handles all `pm_type` dispatching (SPECTATOR→FlyMove, NOCLIP, FREEZE, INTERMISSION, etc.).

### PM_UpdateViewAngles
- **Signature:** `void PM_UpdateViewAngles(playerState_t *ps, const usercmd_t *cmd)`
- **Purpose:** Applies delta_angles to command angles and clamps pitch to ±90°. Public entry for angle-only updates.
- **Inputs:** `ps`, `cmd`
- **Outputs/Return:** Mutates `ps->viewangles`, `ps->delta_angles`.
- **Side effects:** None beyond playerstate.
- **Notes:** No-op during `PM_INTERMISSION` / `PM_SPINTERMISSION`, or when dead (non-spectator).

### PM_WalkMove
- **Signature:** `static void PM_WalkMove(void)`
- **Purpose:** Ground movement: projects wish velocity onto ground plane, applies friction/acceleration, handles duck/wade speed clamping, calls `PM_StepSlideMove`.
- **Inputs:** Global `pm`, `pml`.
- **Side effects:** Modifies `pm->ps->velocity`; may transition to `PM_WaterMove` or `PM_AirMove`.
- **Calls:** `PM_WaterMove`, `PM_CheckJump`, `PM_AirMove`, `PM_Friction`, `PM_CmdScale`, `PM_SetMovementDir`, `PM_ClipVelocity`, `PM_Accelerate`, `PM_StepSlideMove`

### PM_AirMove
- **Signature:** `static void PM_AirMove(void)`
- **Purpose:** Airborne movement with reduced acceleration; projects movement to horizontal plane; clips against steep ground planes.
- **Calls:** `PM_Friction`, `PM_CmdScale`, `PM_SetMovementDir`, `PM_ClipVelocity`, `PM_StepSlideMove`

### PM_Friction
- **Signature:** `static void PM_Friction(void)`
- **Purpose:** Applies ground, water, flight, and spectator friction to `ps->velocity` based on current medium and flags.
- **Side effects:** Directly modifies `pm->ps->velocity`.
- **Notes:** Z-component of velocity is zeroed before speed calculation if walking (ignores slope).

### PM_Accelerate
- **Signature:** `static void PM_Accelerate(vec3_t wishdir, float wishspeed, float accel)`
- **Purpose:** Q2-style velocity acceleration — only adds velocity up to `wishspeed` along `wishdir`.
- **Side effects:** Modifies `pm->ps->velocity`.
- **Notes:** An alternative "proper" implementation (avoiding strafe-jump bug) is present but `#if 0`'d out.

### PM_GroundTrace
- **Signature:** `static void PM_GroundTrace(void)`
- **Purpose:** Casts a short downward trace to determine ground contact, slope validity, landing events, and waterjump cancellation.
- **Side effects:** Sets `pml.groundTrace`, `pml.groundPlane`, `pml.walking`; may call `PM_CrashLand`.
- **Calls:** `pm->trace`, `PM_CorrectAllSolid`, `PM_GroundTraceMissed`, `PM_ForceLegsAnim`, `PM_CrashLand`, `PM_AddTouchEnt`

### PM_Weapon
- **Signature:** `static void PM_Weapon(void)`
- **Purpose:** Manages the full weapon state machine: item use, weapon switching, fire rate (per-weapon `addTime`), ammo deduction, and event generation.
- **Side effects:** Mutates `pm->ps->weapon`, `weaponstate`, `weaponTime`, `ammo[]`; emits weapon events.
- **Calls:** `PM_BeginWeaponChange`, `PM_FinishWeaponChange`, `PM_StartTorsoAnim`, `PM_AddEvent`
- **Notes:** `PW_HASTE` powerup (and MISSIONPACK powerups) divide `addTime` by 1.3/1.5.

### PM_CrashLand
- **Signature:** `static void PM_CrashLand(void)`
- **Purpose:** Calculates landing impact delta using kinematic equations from previous velocity/position; emits fall damage events scaled by water level and duck state.
- **Calls:** `PM_ForceLegsAnim`, `PM_AddEvent`, `PM_FootstepForSurface`

### PM_CheckJump
- **Signature:** `static qboolean PM_CheckJump(void)`
- **Purpose:** Validates jump conditions, sets jump velocity, emits `EV_JUMP`, and forces appropriate leg animation.
- **Side effects:** Modifies `pm->ps->velocity[2]`, `groundEntityNum`, `pm_flags`.

## Control Flow Notes

`Pmove` is the external entry point called each server/client frame. It loops calling `PmoveSingle` until `commandTime` catches up to `cmd.serverTime`. Inside `PmoveSingle`, the sequence is:
1. Setup (clear results, compute `pml.msec`, save previous state, derive view axes)
2. `PM_SetWaterLevel` → `PM_CheckDuck` → `PM_GroundTrace`
3. Movement dispatch (single branch based on `pm_type` and powerup/flag state)
4. `PM_DropTimers` → `PM_Animate` → second `PM_GroundTrace` + `PM_SetWaterLevel`
5. `PM_Weapon` → `PM_TorsoAnimation` → `PM_Footsteps` → `PM_WaterEvents`
6. `trap_SnapVector` on velocity (bandwidth optimization)

## External Dependencies

- **Includes:** `q_shared.h`, `bg_public.h`, `bg_local.h`
- **Defined elsewhere:**
  - `PM_SlideMove`, `PM_StepSlideMove` — defined in `bg_slidemove.c`
  - `BG_AddPredictableEventToPlayerstate` — defined in `bg_misc.c`
  - `trap_SnapVector` — syscall stub; platform-specific (snaps float vector components to integers)
  - `AngleVectors`, `VectorNormalize`, `DotProduct`, etc. — `q_shared.c` / `q_math.c`
  - `bg_itemlist` — defined in `bg_misc.c`
  - `Com_Printf` — engine/qcommon
