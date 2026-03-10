# code/game/bg_slidemove.c

## File Purpose
Implements the sliding collision-response movement for the Quake III pmove system. It resolves player velocity against world geometry by iteratively tracing and clipping velocity along collision planes, and handles automatic step-up over ledges.

## Core Responsibilities
- Trace player movement each frame and clip velocity against hit planes
- Handle up to `MAX_CLIP_PLANES` (5) simultaneous collision planes per move iteration
- Apply gravity interpolation during slide moves
- Detect and resolve two-plane crease collisions via cross-product projection
- Stop the player dead on triple-plane interactions
- Step up over geometry up to `STEPSIZE` (18 units) high via `PM_StepSlideMove`
- Fire step-height events (`EV_STEP_4/8/12/16`) for audio/animation feedback

## Key Types / Data Structures
None defined in this file; relies on types from headers.

## Global / File-Static State
None defined in this file. Accesses `pm` (extern `pmove_t*`) and `pml` (extern `pml_t`) from `bg_local.h`.

## Key Functions / Methods

### PM_SlideMove
- **Signature:** `qboolean PM_SlideMove( qboolean gravity )`
- **Purpose:** Moves the player along a frame timestep, clipping velocity against any planes struck. Up to 4 bump iterations are performed.
- **Inputs:** `gravity` — if `qtrue`, applies gravitational acceleration and averages start/end vertical velocity.
- **Outputs/Return:** `qtrue` if velocity was clipped (i.e., at least one bump occurred); `qfalse` if the full move succeeded unobstructed.
- **Side effects:** Mutates `pm->ps->origin`, `pm->ps->velocity`; updates `pml.impactSpeed`; calls `PM_AddTouchEnt` for each contacted entity; zeroes velocity on `allsolid` or triple-plane lock.
- **Calls:** `pm->trace`, `PM_ClipVelocity`, `PM_AddTouchEnt`, `VectorMA`, `VectorCopy`, `VectorNormalize2`, `VectorClear`, `VectorAdd`, `VectorScale`, `CrossProduct`, `VectorNormalize`, `DotProduct`.
- **Notes:**
  - `MAX_CLIP_PLANES 5` — exceeding this clears velocity and returns early.
  - Duplicate-plane detection (dot > 0.99) nudges velocity outward to avoid epsilon jitter on non-axial planes.
  - If `pm->ps->pm_time` is set, the final velocity is reverted to `primal_velocity` (timer override; marked FIXME in source).
  - `OVERCLIP (1.001f)` prevents re-penetration after clipping.

### PM_StepSlideMove
- **Signature:** `void PM_StepSlideMove( qboolean gravity )`
- **Purpose:** Wraps `PM_SlideMove` with step-up logic: if the flat slide is blocked, attempts to step up by `STEPSIZE` units and re-slides from the elevated position.
- **Inputs:** `gravity` — passed through to `PM_SlideMove`.
- **Outputs/Return:** void; mutates `pm->ps->origin` and `pm->ps->velocity` in place.
- **Side effects:** May fire `PM_AddEvent(EV_STEP_4/8/12/16)` based on delta height. Calls `pm->trace` three times (down-check, up-check, push-down).
- **Calls:** `PM_SlideMove`, `pm->trace`, `PM_ClipVelocity`, `PM_AddEvent`, `VectorCopy`, `VectorSet`, `DotProduct`, `Com_Printf` (debug).
- **Notes:**
  - Step-up is suppressed if the player still has upward velocity and the ground under the start position is steep or absent.
  - The `#if 0` block contains a commented-out "don't step if line-of-sight to start" check.
  - `c_pmove` (extern int) is printed in debug messages.

## Control Flow Notes
Both functions are called from `bg_pmove.c` during the per-frame `Pmove()` update. `PM_StepSlideMove` is the primary entry point used by ground/air locomotion; `PM_SlideMove` is its inner workhorse. These run identically on both client (prediction) and server (authoritative), as the entire `bg_` layer is shared code compiled into both modules.

## External Dependencies
- `q_shared.h` — `vec3_t`, `trace_t`, `qboolean`, vector math macros (`DotProduct`, `VectorMA`, `CrossProduct`, etc.)
- `bg_public.h` — `pmove_t`, `playerState_t`, `EV_STEP_*` event enums, `MAXTOUCH`
- `bg_local.h` — `pml_t`, `STEPSIZE`, `OVERCLIP`, `JUMP_VELOCITY`; extern declarations for `pm`, `pml`, `c_pmove`; declarations of `PM_ClipVelocity`, `PM_AddTouchEnt`, `PM_AddEvent`
- **Defined elsewhere:** `PM_ClipVelocity` (bg_pmove.c), `PM_AddTouchEnt` (bg_pmove.c), `PM_AddEvent` (bg_pmove.c), `pm->trace` callback (set by caller in game/cgame), `Com_Printf` (engine)
