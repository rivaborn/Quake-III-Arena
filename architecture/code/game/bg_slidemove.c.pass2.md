# code/game/bg_slidemove.c — Enhanced Analysis

## Architectural Role

This file implements the **frame-per-frame movement resolver** that bridges per-tick input and world geometry for both the authoritative server (`code/game`) and client prediction (`code/cgame`). As part of the shared `bg_*` layer, `bg_slidemove.c` runs identically on both: the server authorizes player position each tick, while the cgame VM predicts the same movement client-side to show the player their position before server confirmation arrives. The determinism requirement means every collision plane test, gravity application, and velocity clip must produce identical bit-for-bit results across both executables.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/bg_pmove.c`** — Calls `PM_StepSlideMove` as the primary movement integration point during per-frame `Pmove()` in the server game VM
- **`code/cgame/cg_predict.c`** — Calls `PM_StepSlideMove` identically during client-side movement prediction; must stay in sync with server
- Both paths ultimately originate from `PM_Input`/`PM_AirMove`/`PM_GroundMove` logic in `bg_pmove.c`

### Outgoing (what this file depends on)
- **`pm->trace` callback** — Supplied by the hosting module (`code/server/sv_world.c` or `code/cgame/cg_predict.c`); performs BSP collision sweeps via `code/qcommon/cm_trace.c`
- **`PM_ClipVelocity`, `PM_AddTouchEnt`, `PM_AddEvent`** (all in `bg_pmove.c`) — Called for velocity projection, contact tracking, and audio/anim events
- **`pml` singleton** (`code/game/bg_local.h`) — Holds frame state: `groundPlane`, `groundTrace`, `frametime`, `impactSpeed`, `jumpCount`
- **`pm` singleton** (`code/game/bg_local.h`) — Player state: `ps->origin`, `ps->velocity`, `ps->clientNum`, `ps->gravity`, `ps->pm_time`

## Design Patterns & Rationale

**Velocity Clipping via Plane Normals**: The core algorithm projects velocity onto plane normals and reconstructs a new velocity tangent to collisions. This is id-Tech 3 canonical: simple, fast, and works well with BSP geometry.

**Four-Bump Iteration Limit**: Fixed `numbumps = 4` caps the loop iterations. Early Quake engines discovered this prevents edge-case infinite loops without complex termination logic. When `MAX_CLIP_PLANES` is exceeded, the player stops dead to prevent clipping through geometry.

**Epsilon Handling via Duplicate-Plane Nudging** (dot > 0.99): Repeated plane collisions (e.g., corner hits from BSP model cracks) are nudged outward to avoid getting stuck on non-axial surfaces. Prevents the player from "sticking" to geometry between frames.

**Separate Gravity Application**: Gravity is conditionally applied (when `gravity=qtrue`) by averaging initial and final vertical velocity. This decouples movement from physics integration, allowing the same function to handle both ground slides and air movement.

**Step-Up Over Ledges**: Rather than fail when the player clips geometry, `PM_StepSlideMove` attempts to step over obstacles up to `STEPSIZE` (18 units). This "features" small stairs without explicit ramp brushes—a Quake-era design choice that became a gameplay staple.

## Data Flow Through This File

1. **Input**: Caller invokes `PM_StepSlideMove(gravity)` with player state in `pm->ps` and frame delta in `pml.frametime`
2. **Initial Slide** (line 240): `PM_SlideMove(gravity)` attempts unobstructed forward movement
3. **Collision Detection**: Each bump iteration traces an AABB sweep from current origin to desired endpoint (line 105)
4. **Velocity Clipping**: On contact, velocity is projected onto plane normals; multi-plane crease collisions are resolved via cross-product (lines 162–174)
5. **Step-Up** (if needed, line 245–285): If the slide is blocked, re-trace from elevated position and re-slide
6. **Output**: Mutated `pm->ps->origin` and `pm->ps->velocity`; side effects include `pml.impactSpeed` (for fall damage), touch entity tracking, and step-height events (`EV_STEP_4/8/12/16`) for audio

## Learning Notes

**What this teaches about Quake III physics**:
- Movement is *discrete per-frame* (not continuous); frame boundary artifacts are masked by step-up and velocity clipping
- No gravity object or explicit acceleration structure; gravity is a velocity delta applied at frame start
- Traces are AABB only; no ellipsoid/capsule collision (unlike modern engines)
- Impact speed (`pml.impactSpeed`) is used for fall damage; tracked here per-bump for damage detection

**Idiomatic to the era**:
- Fixed bump count (4) is a pragmatic Quake II legacy; no sophisticated convergence logic
- Plane-based velocity clipping is foundational id-Tech 3; seen identically in Quake I, II, III, and JK2
- Triple-plane lock (stop dead) is a corner-case safeguard, not a feature
- Step-up height (18 units) is tuned for Quake's level design scale

**Modern alternatives**:
- Capsule/cylinder sweeps (CCD) instead of discrete AABB traces
- Constraint-based solvers (e.g., PhysX) that integrate gravity as continuous acceleration
- Continuous collision detection to prevent tunneling
- Crease detection via edge/edge sweeps rather than stopping

## Potential Issues

1. **pm_time Override (line 224)**: If `pm->ps->pm_time` is set (e.g., item pickup or knockback), final velocity is reverted to `primal_velocity`. The source comment marks this `FIXME`, suggesting uncertainty about correctness in all scenarios.

2. **Step-Up Suppression**: Line 249–251 suppresses step-up if the player has upward velocity OR the ground is steep. This can create unintuitive behavior on ramps: a player moving upward may fail to step even if there's a low obstacle.

3. **Plane Epsilon (0.99 dot threshold)**: The 0.99 threshold for duplicate plane detection is magic-number tuning. Very small floating-point variations could theoretically cause false positives/negatives on near-parallel planes, though in practice BSP geometry is well-separated.

4. **OVERCLIP Slop (1.001f)**: Clipping velocity by 1.001× prevents re-penetration but can accumulate tiny outward drift over many frames, especially on angled surfaces. Not a practical issue in Q3's fast-paced play, but observable in debug traces.
