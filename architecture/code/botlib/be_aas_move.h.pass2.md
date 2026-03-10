# code/botlib/be_aas_move.h — Enhanced Analysis

## Architectural Role
This header defines the **physics simulation and constraint-checking API** for the AAS subsystem. It serves dual roles: (1) during offline reachability precomputation, it validates whether jump and movement arcs are feasible between areas, and (2) at runtime, it allows bots to query ground/water states and predict movement trajectories for planning. The movement prediction functions are the computational core of the reachability graph—without them, the engine cannot determine which areas are reachable from others.

## Key Cross-References

### Incoming (who depends on this)
- **be_aas_reach.c** (`AAS_InitReachability`): calls `AAS_PredictClientMovement`, `AAS_ClientMovementHitBBox`, `AAS_RocketJumpZVelocity`, `AAS_BFGJumpZVelocity`, `AAS_HorizontalVelocityForJump`, `AAS_OnGround`, `AAS_Swimming` during reachability link validation
- **be_aas_route.c**: calls `AAS_PredictRoute` which internally uses movement prediction for alt-routing fallbacks
- **be_aas_routealt.c**: uses movement prediction for alternative pathing around blocked routes
- **Game VM via trap_BotLib***: bot movement AI invokes these functions indirectly through botlib's EA (elementary action) layer

### Outgoing (what this depends on)
- **be_aas_bspq3.c** (`AAS_Trace`): the movement prediction implementation traces against BSP geometry and entity collisions to detect ground/wall contact
- **be_aas_sample.c** (`AAS_PointAreaNum`, `AAS_TraceAreas`, `AAS_PresenceTypeBoundingBox`): area queries and bounding-box sweeps used during movement stepping
- **be_aas_def.h**: struct definitions (`aas_clientmove_s`, `aas_reachability_s`)
- **aassettings** global: physics tuning (gravity, friction, step height, jump force) encapsulated within AASINTERN scope only

## Design Patterns & Rationale

**Physics Sandbox Pattern**: The movement prediction API decouples physics simulation from game state mutation. Reachability precomputation can safely test thousands of hypothetical movement arcs without side effects. This is critical because reachability computation is expensive (potentially 100,000s of traces per map) and must remain deterministic.

**Specialized Predictor for Target-Hit Testing**: `AAS_ClientMovementHitBBox` is a variant of the main predictor that terminates early when the client bounds intersect a target AABB. This avoids full-trajectory simulation when only collision detection is needed—an optimization for jump reachability validation.

**Weapon-Specific Physics**: `AAS_RocketJumpZVelocity` and `AAS_BFGJumpZVelocity` abstract weapon-blast kinematics into reusable functions. This allows reachability computation to consider weapon-assisted jumps as valid movement modes (travel types `TRAVEL_ROCKETJUMP`, `TRAVEL_BFGJUMP`).

**Inverse Kinematic Solver**: `AAS_HorizontalVelocityForJump` solves the projectile arc equation given a desired Z-velocity and start/end positions. This is essential for computing whether a jump from area A can reach area B—it finds the required run speed.

**Encapsulation via AASINTERN**: The global `aassettings` is guarded by a preprocessor gate, visible only to internal AAS modules. This prevents external code from accidentally reading stale physics settings, but also means tuning requires recompilation (no runtime parameter override capability).

## Data Flow Through This File

**Precomputation Pipeline** (offline, in bspc):
1. For each area pair, reachability logic calls `AAS_PredictClientMovement(origin_A, velocity_toward_B, ...)`
2. Movement simulation traces each frame, accumulating position until target area is reached or obstacle hit
3. If collision occurs, the reachability link is marked infeasible; otherwise, link is cached as valid
4. Weapon jump functions (`AAS_RocketJumpZVelocity`, `AAS_HorizontalVelocityForJump`) validate special travel types

**Runtime Bot Movement** (in-game):
1. Bot AI calls movement prediction with current bot state (position, velocity, desired movement direction)
2. Renderer optionally visualizes trajectory if `visualize=1` (debug aid)
3. Result struct contains final predicted position, velocity, events (landed, entered water, hit wall)
4. Bot pathfinding uses this to refine movement commands each frame

**Constraint Checking** (both phases):
- `AAS_OnGround`, `AAS_Swimming`, `AAS_AgainstLadder` perform single-point queries to classify environment
- Used to gate movement type transitions (can only jump if grounded, can only climb if against ladder)

## Learning Notes

**Physics Abstraction Era**: This code reflects early 2000s physics design—deterministic, frame-by-frame stepping with no constraint solvers. Modern engines use continuous collision detection and impulse-based resolution. The Quake III approach trades accuracy for simplicity and predictability.

**Separation of Concerns**: Notice that the header exposes only the physics API, not the implementation details. The actual gravity/friction logic lives in be_aas_move.c (`AAS_Accelerate`, `AAS_ApplyFriction`), keeping the contract clean.

**Precomputation-Driven Navigation**: Unlike modern pathfinding (which may compute paths on-demand), Quake III bakes reachability links into the AAS file during offline compilation. This header defines the *validators* that justify those links—every link in the final AAS graph has been vetted by movement prediction.

**Vector Math Over Formal Kinematics**: The jump arc solver (`AAS_HorizontalVelocityForJump`) uses algebraic geometry rather than physics integration—it solves parabolic equations directly, which is both faster and more precise for validation.

## Potential Issues

- **Long Function Signature**: `AAS_PredictClientMovement` takes 12 parameters, violating modern best practices. A parameter struct would improve readability and allow future extensibility without breaking the ABI.
- **Global Mutation**: Movement prediction may invoke debug visualization through engine callbacks, creating hidden coupling to the renderer subsystem. The `visualize` flag isn't documented as to thread-safety or performance cost.
- **No Error Propagation**: Functions return success/failure but don't distinguish failure causes (invalid input, out-of-bounds prediction, etc.), making debugging harder.
- **Physics Tuning Opacity**: `aassettings` is opaque to non-AAS modules. If a bot behaves unexpectedly, engineers must grep botlib source to find where gravity/friction are defined; no external visibility or introspection API exists.
