# code/botlib/be_aas_move.c — Enhanced Analysis

## Architectural Role

This file implements the **deterministic physics simulation engine** for the AAS (Area Awareness System) navigation subsystem in botlib. It sits at the core of bot reachability validation: by accurately predicting whether a movement (jump, fall, strafe, etc.) will succeed from one area to another without executing it, bots can make cost-effective offline pathfinding decisions. The file bridges AAS geometric queries (point-to-area mapping, BBox traces) with physics constants tuned to match the server's authoritative `bg_pmove.c` player movement, ensuring bot predictions are accurate. This is essential for both **reachability link computation** (during AAS file generation) and **runtime movement AI** (during bot thinking).

## Key Cross-References

### Incoming (who depends on this file)

| Caller | Usage |
|--------|-------|
| `code/botlib/be_aas_reach.c` | Calls `AAS_PredictClientMovement` to validate jump/teleport/ladder reachabilities during link computation; calls `AAS_HorizontalVelocityForJump` to solve for required horizontal speed |
| `code/botlib/be_ai_move.c` | Calls movement prediction to evaluate navigation options during per-frame bot AI thinking |
| `code/bspc/be_aas_bspc.c` (offline compiler) | Calls `AAS_InitSettings` to load physics parameters before reachability computation |
| `code/botlib/be_aas_sample.c` | Calls state predicates (`AAS_OnGround`, `AAS_Swimming`, `AAS_AgainstLadder`) to classify areas and query point properties |
| Global `aassettings` | Read by `be_aas_reach.c`, `be_aas_route.c` for physics-dependent distance weighting and risk scoring |

### Outgoing (what this file depends on)

| Dependency | Used For |
|------------|----------|
| `code/botlib/be_aas_sample.c` | `AAS_TraceClientBBox`, `AAS_TraceAreas`, `AAS_PointAreaNum`, `AAS_PointContents`, `AAS_PointPresenceType`, `AAS_PresenceTypeBoundingBox` — translate between world-space movement and area topology |
| `code/botlib/be_aas_bspq3.c` | `AAS_Trace`, `AAS_PlaneFromNum` — BSP collision and geometry queries |
| `code/qcommon/cm_*.c` (via import) | Indirectly: collision model traces passed through `botimport` callback |
| Global `aasworld` | Reads area/face/plane array data during traces and state checks |
| Math library (`q_shared.c`) | `DotProduct`, `VectorNormalize`, `AngleVectors`, `VectorMA`, `VectorScale`, `VectorLength` — standard 3D ops |
| `code/botlib/be_aas_debug.c` (optional) | `AAS_DebugLine`, `AAS_ClearShownDebugLines` for visualization (only if `visualize=1`) |

## Design Patterns & Rationale

**Deterministic Physics Simulation:**  
The file does not integrate with the game VM or frame loop; instead, it runs a **self-contained discrete-time physics engine** that can be invoked on-demand. This allows bots to "fast-forward" 10–100 frames of movement in a single call, validating whether a reachability is achievable without waiting for actual gameplay. Parameters (gravity, friction, max speed, acceleration) are loaded from cvars, matching the server's authoritative physics, so predictions are guaranteed accurate (modulo BSP geometry queries).

**Step-Over Detection as Special Case:**  
When the bot hits a wall horizontally but hasn't jumped recently, the code performs a "step detection" inner loop: it tries to step **up** the wall and continue. This models the automatic step-over behavior in the Quake movement engine, avoiding unnecessary reachability links for low walls. The 20-iteration limit prevents infinite loops on complex geometry.

**Early Termination via Stop Events:**  
Rather than always simulating to `maxframes`, the prediction stops on the first triggered event (ground contact, liquid entry, gap, bounding-box collision, area change). This makes prediction cheap: validating most reachabilities takes only 1–3 frames. Bot AI code can mask events to control when to stop.

**Wrapper Pattern with Unsafe Defaults:**  
`AAS_PredictClientMovement` (public wrapper) allocates uninitialized stack `mins`/`maxs`, then calls the internal `AAS_ClientMovementPrediction`. This is safe only if `stopevent` does not include `SE_HITBOUNDINGBOX`; otherwise, the uninitialized bounding box is used for collision. The safer variant `AAS_ClientMovementHitBBox` takes explicit AABB parameters. This design avoids breaking the API for callers who don't need bounding-box collision.

## Data Flow Through This File

```
Bot AI (game VM)
    ↓
AAS_PredictClientMovement(origin, velocity, cmdmove, ..., stopevent)
    ↓
AAS_ClientMovementPrediction(...)
    ↓ [per frame loop, up to maxframes]
    ├─ AAS_TraceClientBBox() → movement collision with world
    ├─ AAS_Accelerate() → apply command input
    ├─ AAS_ApplyFriction() → drag
    ├─ [step detection loop] → try to step over obstacles
    ├─ AAS_PointContents() → check for liquid
    ├─ AAS_OnGround() → detect ground contact
    └─ [check stop conditions]
    ↓
Output: aas_clientmove_t
    ├─ endpos, endvelocity (final state)
    ├─ stopevent (why prediction stopped)
    ├─ area (final area number)
    └─ time, frames (elapsed simulation time)
    ↓
Bot AI uses result to decide: "Is this reachability achievable?"
```

**Typical usage in reachability validation** (`be_aas_reach.c`):
1. Predict from jump start, initial upward velocity, target area
2. If `stopevent & SE_ENTERLAVA`, mark reachability as dangerous
3. If final area matches target, reachability is valid
4. If final position overshoots, maybe require strafe air-control

## Learning Notes

**Quake Physics Idiosyncrasies:**  
This file encodes the specific movement model of Quake III: arcade-style (no realistic friction or air resistance), with automatic step-over, strafe air-control, and self-damage jump mechanics. Modern game engines use more realistic physics engines (Havok, PhysX); Q3 movement is hand-tuned and deterministic, essential for both network synchronization and bot AI.

**Physics Constants as Cvars:**  
All physics parameters are **loaded from library variables at init time** (`AAS_InitSettings`). This makes bot movement completely tunable without recompilation—level designers can adjust `phys_gravity`, `phys_jumpvel`, etc., and bots will adapt automatically. This is a form of **data-driven design** that Q3 pioneered.

**Movement Prediction ≠ Client Prediction:**  
Do not confuse `AAS_PredictClientMovement` (used here for offline bot reasoning) with client-side prediction in cgame (`CG_PredictPlayerState` in `cg_predict.c`). Both simulate `Pmove`, but:
- **Client prediction** accumulates unacknowledged input and visually interpolates the player, correcting on server snapshot
- **AAS prediction** is single-threaded, non-interactive, used to evaluate the *possibility* of a move

**Step Limiting & Robustness:**  
The 20-iteration step-detection loop and maxframes-limited outer loop are defensive: they prevent bot simulations from hanging the server if geometry is degenerate. This is essential in shipping code; modern engines use physics substeps with limits as well.

**Weapon Jump Calculation:**  
`AAS_WeaponJumpZVelocity` models the **rocket self-damage knockback** mechanic—a signature Q3 move. It simulates:
1. Bot looks downward and shoots a rocket
2. Traces the impact point
3. Calculates damage falloff with distance
4. Applies knockback physics (acceleration ∝ damage / mass)
5. Returns the resulting Z velocity

This is highly specialized: bots can predict whether a rocket jump can reach a high platform, enabling complex routes inaccessible by normal movement. No modern casual engine game exposes this level of physics manipulation to bots.

## Potential Issues

**Issue 1: Uninitialized Bounding Box in Public Wrapper**  
`AAS_PredictClientMovement` allocates stack-local `mins`/`maxs` that are never initialized, then passes them to `AAS_ClientMovementPrediction`. If a caller includes `SE_HITBOUNDINGBOX` in `stopevent`, the function will perform collision tests against garbage memory values. **Mitigation:** Callers must ensure they never set this flag, or use `AAS_ClientMovementHitBBox` instead. This is documented only via naming convention, not enforced.

**Issue 2: Step Detection Loop Limit**  
The inner step-detection loop has an explicit 20-iteration limit, but there is no warning or logging if this limit is reached. On complex or degenerate geometry, stepping might terminate prematurely without the caller realizing, leading to silent reachability validation errors. A production engine would log a warning or cap the iteration count more conservatively.

**Issue 3: Frametime Hardcoding in Some Callers**  
The default frametime in `AAS_JumpReachRunStart` is hardcoded to `0.1f` seconds per frame. If the server's tickrate differs from the assumption (e.g., higher client tickrate or variable timestep), the predicted trajectories will diverge from the actual server physics, causing bots to mis-estimate feasibility. Q3's fixed 125 Hz tickrate makes this safe, but it's a brittleness point.

**Issue 4: No Interaction with Dynamic Entities**  
Movement prediction uses only BSP collision (`AAS_TraceClientBBox`). Dynamic movers (doors, elevators, func_bob) are traced at prediction time, but moving targets or time-dependent geometry (e.g., platforms that appear/disappear) are not modeled. Reachabilities across moving platforms may be validated pessimistically (assume the platform is in an unfavorable position).
