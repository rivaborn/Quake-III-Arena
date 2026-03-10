# code/botlib/be_aas_move.h

## File Purpose
Public header for the AAS (Area Awareness System) movement prediction subsystem within the Quake III botlib. It declares functions used to simulate and predict client movement physics for bot navigation, including ground checks, swimming, ladder detection, and weapon-assisted jumping.

## Core Responsibilities
- Expose movement prediction API (`AAS_PredictClientMovement`, `AAS_ClientMovementHitBBox`)
- Provide terrain/environment query utilities (ground, water, ladder detection)
- Expose weapon-jump velocity calculators (rocket jump, BFG jump)
- Declare jump arc/trajectory helpers for reachability computation
- Conditionally expose internal `aassettings` global to other AAS modules

## Key Types / Data Structures
None defined here; forward-references structs defined elsewhere.

| Name | Kind | Purpose |
|---|---|---|
| `aas_clientmove_s` | struct (forward ref) | Result of a movement prediction step; defined in `be_aas_def.h` |
| `aas_reachability_s` | struct (forward ref) | AAS reachability link between areas; defined in `be_aas_def.h` |
| `aas_settings_t` | typedef (extern ref) | Global AAS configuration/physics settings |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `aassettings` | `aas_settings_t` | global (guarded by `AASINTERN`) | AAS physics and tuning settings; only visible to internal AAS translation units |

## Key Functions / Methods

### AAS_PredictClientMovement
- **Signature:** `int AAS_PredictClientMovement(struct aas_clientmove_s *move, int entnum, vec3_t origin, int presencetype, int onground, vec3_t velocity, vec3_t cmdmove, int cmdframes, int maxframes, float frametime, int stopevent, int stopareanum, int visualize)`
- **Purpose:** Simulates client physics forward in time, stepping frame-by-frame, populating `move` with final state.
- **Inputs:** Entity number, start origin/velocity, presence type (crouch/normal), ground state, command movement vector, frame counts, stop conditions, optional visualization flag.
- **Outputs/Return:** Non-zero on success; `move` struct populated with predicted position/velocity/events.
- **Side effects:** May invoke debug rendering if `visualize` is set.
- **Calls:** Defined in `be_aas_move.c`; calls internal trace/physics routines.
- **Notes:** `stopevent` and `stopareanum` allow early-exit prediction (e.g., stop when entering a specific area or hitting water).

### AAS_ClientMovementHitBBox
- **Signature:** `int AAS_ClientMovementHitBBox(struct aas_clientmove_s *move, int entnum, vec3_t origin, int presencetype, int onground, vec3_t velocity, vec3_t cmdmove, int cmdframes, int maxframes, float frametime, vec3_t mins, vec3_t maxs, int visualize)`
- **Purpose:** Variant of movement prediction that stops when the client bounding box intersects a specified AABB.
- **Inputs:** Same as `AAS_PredictClientMovement` plus `mins`/`maxs` defining the target bounding box.
- **Outputs/Return:** Non-zero if bbox was hit; `move` populated with state at collision.
- **Side effects:** Optional visualization.
- **Notes:** Used for checking reachability — e.g., will the bot land inside a target volume?

### AAS_OnGround
- **Signature:** `int AAS_OnGround(vec3_t origin, int presencetype, int passent)`
- **Purpose:** Tests whether a client at `origin` is resting on solid ground.
- **Inputs:** World position, presence type, entity to pass through during trace.
- **Outputs/Return:** Non-zero if on ground.
- **Side effects:** Issues a downward trace against world geometry.

### AAS_Swimming
- **Signature:** `int AAS_Swimming(vec3_t origin)`
- **Purpose:** Returns true if the point `origin` is inside a water volume.
- **Inputs:** World position.
- **Outputs/Return:** Non-zero if in water.

### AAS_RocketJumpZVelocity / AAS_BFGJumpZVelocity
- **Signature:** `float AAS_RocketJumpZVelocity(vec3_t origin)` / `float AAS_BFGJumpZVelocity(vec3_t origin)`
- **Purpose:** Compute the upward Z velocity a bot would gain from a rocket/BFG self-blast at `origin`, used to determine if a weapon-jump reachability is feasible.
- **Inputs:** Origin of the jump.
- **Outputs/Return:** Z velocity (float) imparted by weapon blast.

### AAS_HorizontalVelocityForJump
- **Signature:** `int AAS_HorizontalVelocityForJump(float zvel, vec3_t start, vec3_t end, float *velocity)`
- **Purpose:** Solves projectile arc equations to find required horizontal speed to reach `end` from `start` given a known vertical velocity `zvel`.
- **Inputs:** Vertical velocity, start/end world positions.
- **Outputs/Return:** Non-zero if a valid solution exists; horizontal speed written to `*velocity`.
- **Notes:** Used during reachability precomputation to validate jump arcs.

### AAS_JumpReachRunStart
- **Signature:** `void AAS_JumpReachRunStart(struct aas_reachability_s *reach, vec3_t runstart)`
- **Purpose:** Computes the point where a bot should begin running before a jump reachability to gain enough speed.
- **Outputs/Return:** `runstart` populated with world position.

### Notes
- `AAS_AgainstLadder`, `AAS_SetMovedir`, `AAS_DropToFloor`, and `AAS_InitSettings` are utility/init helpers with straightforward purposes (ladder probe, angle-to-direction conversion, floor snap, settings initialization).

## Control Flow Notes
This header is included by both internal AAS modules (with `AASINTERN` defined, gaining access to `aassettings`) and external consumers such as `be_aas_reach.c` and `be_aas_route.c` during reachability precomputation. At runtime, `AAS_PredictClientMovement` is called by the bot movement AI each frame to plan safe paths.

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `aas_clientmove_s`, `aas_reachability_s`, `aas_settings_t` — defined in `be_aas_def.h`
- `AASINTERN` — preprocessor guard defined by internal AAS translation units only
