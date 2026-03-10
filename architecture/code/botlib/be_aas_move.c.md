# code/botlib/be_aas_move.c

## File Purpose
Implements AAS (Area Awareness System) movement physics simulation for the Quake III bot library. It predicts client movement trajectories by simulating gravity, friction, acceleration, stepping, and liquid content detection. Results are used by the bot AI to evaluate reachability and plan navigation.

## Core Responsibilities
- Initialize AAS physics settings from library variables (`aassettings`)
- Detect ground contact, ladder proximity, and swimming state
- Simulate multi-frame client movement with full physics (gravity, friction, acceleration, stepping, crouching, jumping)
- Report movement stop-events (hit ground, enter liquid, enter area, fall damage, gap, bounding-box collision)
- Calculate horizontal velocity required for a jump arc between two points
- Calculate Z-velocity resulting from rocket/BFG self-damage jumps

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_settings_t` | struct (extern, defined in `be_aas_def.h`) | Physics constants (gravity, friction, speed caps, jump velocity, risk scores) used throughout movement simulation |
| `aas_clientmove_t` | struct (extern) | Output of movement prediction: end position, velocity, stop event, area, presence type, elapsed time/frames |
| `aas_trace_t` | struct (extern) | Result of an AAS bounding-box trace: fraction, endpos, startsolid, planenum, area |
| `bsp_trace_t` | struct (extern) | BSP-level trace result used for weapon-jump calculations |
| `aas_reachability_t` | struct (extern) | Reachability record; used here to compute jump run-up start positions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aassettings` | `aas_settings_t` | global | Physics parameters loaded at init; read by all movement functions |
| `botimport` | `botlib_import_t` (extern) | global | Engine import table; used for debug printing |
| `VEC_UP`, `MOVEDIR_UP`, `VEC_DOWN`, `MOVEDIR_DOWN` | `vec3_t` (static) | file-static | Sentinel angle vectors for `AAS_SetMovedir` special-case directions |

## Key Functions / Methods

### AAS_InitSettings
- **Signature:** `void AAS_InitSettings(void)`
- **Purpose:** Populates `aassettings` with physics and risk-score values read from botlib library variables (cvars).
- **Inputs:** None (reads from `LibVarValue`)
- **Outputs/Return:** None; modifies `aassettings` global
- **Side effects:** Writes all fields of the `aassettings` global
- **Calls:** `LibVarValue` (×38)
- **Notes:** Called once at AAS initialization. Defaults reflect Q3 player physics constants.

### AAS_DropToFloor
- **Signature:** `int AAS_DropToFloor(vec3_t origin, vec3_t mins, vec3_t maxs)`
- **Purpose:** Snaps an origin downward 100 units to the nearest solid floor via BSP trace.
- **Inputs:** `origin` (in/out), `mins`/`maxs` bounding box
- **Outputs/Return:** `qtrue` on success; modifies `origin` in place
- **Side effects:** None
- **Calls:** `AAS_Trace`

### AAS_AgainstLadder
- **Signature:** `int AAS_AgainstLadder(vec3_t origin)`
- **Purpose:** Determines whether the given position is flush against a ladder face in the AAS world.
- **Inputs:** `origin`
- **Outputs/Return:** `qtrue` if against a ladder face, `qfalse` otherwise
- **Side effects:** None
- **Calls:** `AAS_PointAreaNum`, `AAS_PointInsideFace`, `DotProduct`
- **Notes:** Attempts up to 5 jittered origins to escape solid areas before giving up.

### AAS_OnGround
- **Signature:** `int AAS_OnGround(vec3_t origin, int presencetype, int passent)`
- **Purpose:** Returns whether the entity is standing on non-steep ground within 10 units below origin.
- **Inputs:** `origin`, presence type, passent entity number
- **Outputs/Return:** `qtrue` / `qfalse`
- **Side effects:** None
- **Calls:** `AAS_TraceClientBBox`, `AAS_PlaneFromNum`, `DotProduct`

### AAS_Swimming
- **Signature:** `int AAS_Swimming(vec3_t origin)`
- **Purpose:** Returns whether the point 2 units below origin is inside water, slime, or lava.
- **Inputs:** `origin`
- **Outputs/Return:** `qtrue` / `qfalse`
- **Side effects:** None
- **Calls:** `AAS_PointContents`

### AAS_WeaponJumpZVelocity
- **Signature:** `float AAS_WeaponJumpZVelocity(vec3_t origin, float radiusdamage)`
- **Purpose:** Computes the upward Z velocity a bot would receive from a self-inflicted weapon blast (rocket/BFG jump) at the given origin.
- **Inputs:** `origin`, `radiusdamage` (weapon-specific)
- **Outputs/Return:** Resulting Z velocity (float)
- **Side effects:** None
- **Calls:** `AAS_Trace`, `AngleVectors`, `VectorMA`, `VectorLength`, `VectorNormalize`
- **Notes:** Simulates shooting straight down, traces the impact point, calculates knockback damage physics, then adds jump velocity.

### AAS_Accelerate
- **Signature:** `void AAS_Accelerate(vec3_t velocity, float frametime, vec3_t wishdir, float wishspeed, float accel)`
- **Purpose:** Applies Quake 2-style acceleration to a velocity vector.
- **Inputs:** `velocity` (in/out), `frametime`, desired direction/speed, acceleration scalar
- **Outputs/Return:** Modifies `velocity` in place
- **Side effects:** None
- **Calls:** `DotProduct`

### AAS_ApplyFriction
- **Signature:** `void AAS_ApplyFriction(vec3_t vel, float friction, float stopspeed, float frametime)`
- **Purpose:** Reduces horizontal velocity components by friction each frame.
- **Inputs:** `vel` (in/out), friction coefficient, stop-speed threshold, frame delta
- **Outputs/Return:** Modifies `vel` in place (XY only)
- **Side effects:** None

### AAS_ClipToBBox
- **Signature:** `int AAS_ClipToBBox(aas_trace_t *trace, vec3_t start, vec3_t end, int presencetype, vec3_t mins, vec3_t maxs)`
- **Purpose:** Clips a movement segment against an axis-aligned bounding box (Minkowski sum with presence type).
- **Inputs:** Trace output pointer, start/end positions, presence type, target AABB
- **Outputs/Return:** `qtrue` if collision occurred; fills `trace`
- **Side effects:** Writes to `*trace`
- **Calls:** `AAS_PresenceTypeBoundingBox`

### AAS_ClientMovementPrediction
- **Signature:** `int AAS_ClientMovementPrediction(struct aas_clientmove_s *move, int entnum, vec3_t origin, int presencetype, int onground, vec3_t velocity, vec3_t cmdmove, int cmdframes, int maxframes, float frametime, int stopevent, int stopareanum, vec3_t mins, vec3_t maxs, int visualize)`
- **Purpose:** Core multi-frame physics simulation loop. Simulates up to `maxframes` frames of player movement and returns when a `stopevent` condition triggers or frames are exhausted.
- **Inputs:** Full movement state (origin, velocity, presence, command input), prediction horizon, stop conditions, optional AABB for `SE_HITBOUNDINGBOX`
- **Outputs/Return:** `qtrue`; populates `*move` with end position, area, velocity, triggering event, and elapsed time
- **Side effects:** Reads `aasworld` global; calls `botimport.Print` if `visualize` and start-solid; calls `AAS_DebugLine` for visualization
- **Calls:** `AAS_Swimming`, `AAS_ApplyFriction`, `AAS_Accelerate`, `AAS_TraceClientBBox`, `AAS_TraceAreas`, `AAS_ClipToBBox`, `AAS_PlaneFromNum`, `AAS_PointAreaNum`, `AAS_OnGround`, `AAS_PointContents`, `AAS_PointPresenceType`, `AAS_DebugLine`
- **Notes:** Step detection is performed for vertical wall impacts when the bot hasn't jumped recently. Inner loop limited to 20 iterations to prevent infinite loops.

### AAS_PredictClientMovement
- **Signature:** `int AAS_PredictClientMovement(..., int visualize)`
- **Purpose:** Public wrapper around `AAS_ClientMovementPrediction`; does not accept explicit bounding box (uses uninitialized stack mins/maxs — only valid when `stopevent` does not include `SE_HITBOUNDINGBOX`).
- **Calls:** `AAS_ClientMovementPrediction`

### AAS_ClientMovementHitBBox
- **Signature:** `int AAS_ClientMovementHitBBox(..., vec3_t mins, vec3_t maxs, int visualize)`
- **Purpose:** Public wrapper that forces `SE_HITBOUNDINGBOX` stop event with caller-supplied AABB.
- **Calls:** `AAS_ClientMovementPrediction`

### AAS_HorizontalVelocityForJump
- **Signature:** `int AAS_HorizontalVelocityForJump(float zvel, vec3_t start, vec3_t end, float *velocity)`
- **Purpose:** Computes horizontal speed needed to travel from `start` to `end` given an initial Z velocity using projectile motion equations.
- **Inputs:** Initial Z velocity, start/end positions, output pointer
- **Outputs/Return:** `1` if feasible (speed ≤ `phys_maxvelocity`), `0` if target is too high or too far; sets `*velocity`
- **Side effects:** None
- **Notes:** Returns 0 with `phys_maxvelocity` if time denominator is zero to avoid division by zero.

- **Notes (minor functions):** `AAS_SetMovedir` maps special UP/DOWN sentinel angles to world-space unit vectors. `AAS_JumpReachRunStart` uses a short prediction to find a safe run-up start before a jump reachability. `AAS_RocketJumpZVelocity` / `AAS_BFGJumpZVelocity` delegate to `AAS_WeaponJumpZVelocity` with hardcoded radius values. `AAS_TestMovementPrediction` is a debug-only entry point.

## Control Flow Notes
Called during bot AI thinking (not every render frame). `AAS_InitSettings` is an init-phase function. `AAS_PredictClientMovement` and `AAS_ClientMovementHitBBox` are called from reachability analysis (`be_aas_reach.c`) and movement AI (`be_ai_move.c`) to evaluate navigation options. No shutdown logic is present in this file.

## External Dependencies
- `../game/q_shared.h` — math types, vector macros, `qboolean`
- `l_libvar.h` — `LibVarValue` for reading physics cvars
- `be_aas_funcs.h` — `AAS_Trace`, `AAS_TraceClientBBox`, `AAS_TraceAreas`, `AAS_PointAreaNum`, `AAS_PointContents`, `AAS_PlaneFromNum`, `AAS_PointPresenceType`, `AAS_PresenceTypeBoundingBox`, `AAS_PointInsideFace`
- `be_aas_def.h` — `aasworld` global, `aas_settings_t`, area/face/plane data structures, flag constants (`AREA_LADDER`, `FACE_LADDER`, `PRESENCE_*`, `AREACONTENTS_*`, `SE_*`)
- `../game/botlib.h` — `botlib_import_t`, `botimport` (print/debug I/O)
- `be_aas_debug.h` (implicit via funcs) — `AAS_DebugLine`, `AAS_ClearShownDebugLines` (defined elsewhere)
- `AngleVectors`, `VectorNormalize`, `DotProduct`, `Com_Memset` — defined in `q_shared.c` / math libraries
