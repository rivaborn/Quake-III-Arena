# code/cgame/cg_predict.c — Enhanced Analysis

## Architectural Role

This file is the **prediction core** of the cgame VM, sitting at the intersection of three data streams: server-delivered `snapshot_t` frames, the client-side `usercmd_t` ring buffer, and the shared `bg_pmove.c` physics engine. It is the primary reason cgame and game VMs must compile `bg_pmove.c`/`bg_misc.c` from identical sources: client-side `Pmove` must produce bit-identical results to the server's authoritative run to minimize prediction error. `cg_predict.c` also serves as the **collision abstraction layer** for the cgame VM, providing `CG_Trace` and `CG_PointContents` wrappers that unify world-BSP queries (via `trap_CM_*` syscalls through to `qcommon/cm_trace.c`) with entity-level AABB and bmodel sweeps.

## Key Cross-References

### Incoming (who depends on this file)

- **`cg_view.c` → `CG_PredictPlayerState`**: Called once per rendered frame from `CG_DrawActiveFrame`, the top-level cgame render entry point. This is the only caller; it drives the entire prediction pipeline.
- **`cg_snapshot.c` → `CG_BuildSolidList`**: Called whenever `cg.snap` is advanced (new server snapshot ingested). The solid/trigger lists must be rebuilt before any trace in the upcoming frame is valid.
- **`bg_pmove.c` → `cg_pmove.trace` / `cg_pmove.pointcontents`**: `Pmove` invokes these exclusively through the function pointers injected here — it never calls into the collision system directly. This is the engine's dependency injection seam.
- **`cg_ents.c` → `CG_Trace`** (via `CG_AdjustPositionForMover`): The mover correction step reads from `CG_Trace` indirectly. `CG_TransitionPlayerState` in `cg_playerstate.c` is also driven by `CG_PredictPlayerState` after each `Pmove` step.

### Outgoing (what this file depends on)

- **`bg_pmove.c` (`Pmove`, `PM_UpdateViewAngles`, `BG_TouchJumpPad`, `BG_EvaluateTrajectory`, `BG_AddPredictableEventToPlayerstate`, `BG_PlayerTouchesItem`, `BG_CanItemBeGrabbed`)**: The shared game/cgame physics layer. Any divergence from the server's `bg_pmove.c` build produces visible misprediction.
- **`qcommon` via syscall layer (`cg_syscalls.c`)**: All `trap_CM_*` calls cross the VM boundary into `qcommon/cm_trace.c` and `cm_test.c`. The cgame VM has no direct access to collision structures.
- **`cg_main.c` globals** (`cg`, `cgs`, `cg_entities[]`, `cg_predictItems`, `cg_nopredict`, `cg_synchronousClients`, `cg_showmiss`, `pmove_fixed`, `pmove_msec`, `cg_errorDecay`): Nearly all per-frame state flows through the global `cg` struct.
- **`cg_ents.c` (`CG_AdjustPositionForMover`)** and **`cg_playerstate.c` (`CG_TransitionPlayerState`)**: Post-`Pmove` correction and event firing, respectively.

## Design Patterns & Rationale

**Dependency injection via function pointers**: `cg_pmove.trace` and `cg_pmove.pointcontents` are assigned `CG_Trace`/`CG_PointContents` before every prediction run. This allows `Pmove` (compiled into both game and cgame VMs) to remain agnostic about which collision backend it uses — on the server it calls `trap_Trace`; here it calls through the cgame wrappers that additionally test client-side entity geometry.

**Pre-filtered entity sublists**: `CG_BuildSolidList` partitions entities into solid and trigger arrays once per snapshot. This converts an O(n_entities) scan from every `CG_ClipMoveToEntities` call into an O(n_solid) inner loop, critical given that `Pmove` may issue multiple traces per step per frame.

**Full command replay (not delta)**: Every unacknowledged `usercmd_t` from `current - CMD_BACKUP + 1` to `current` is re-simulated each frame. This is intentionally redundant — it handles packet loss gracefully at the cost of CPU on high-latency connections. The in-code `OPTIMIZE` comment acknowledges an obvious improvement (cache intermediate `playerState_t`) that was never implemented.

**Smooth error decay**: When the server's acknowledged `playerState_t` diverges from the predicted one, the delta is stored in `cg.predictedError` and bled off over `cg_errorDecay` milliseconds. This trades correctness for visual smoothness, an explicit UX tradeoff.

## Data Flow Through This File

```
Server snapshot (cg.snap / cg.nextSnap)
    │
    ├─ CG_BuildSolidList() ──→ cg_solidEntities[], cg_triggerEntities[]
    │
    └─ CG_PredictPlayerState()
           │
           ├─ [demo/follow/nopredict] → CG_InterpolatePlayerState()
           │       lerps ps between two snapshots, optionally grabs live angles
           │
           └─ [normal gameplay] →
                  seed predictedPlayerState from nextSnap (or snap)
                  │
                  for each unacknowledged usercmd_t:
                  │   Pmove(&cg_pmove)   ←── CG_Trace / CG_PointContents
                  │       │                       │
                  │       │              trap_CM_BoxTrace (world BSP)
                  │       │            + CG_ClipMoveToEntities (solid ents)
                  │       │
                  │   CG_TouchTriggerPrediction()
                  │       ├── CG_TouchItem() → modify predictedPlayerState
                  │       └── BG_TouchJumpPad / set cg.hyperspace
                  │
                  CG_AdjustPositionForMover()
                  CG_TransitionPlayerState() → fire events
                  │
                  decay cg.predictedError
                  │
                  ▼
           cg.predictedPlayerState  (consumed by cg_view.c for camera, cg_draw.c for HUD)
```

## Learning Notes

- **Shared physics for determinism**: The pattern of compiling `bg_pmove.c` identically into both game and cgame VMs is Q3's solution to the classic "client prediction" problem. Any platform-specific floating-point behavior differences between VM builds will cause persistent prediction error — this is why Q3 clamps `pmove_msec` to a fixed range and exposes `pmove_fixed` for deterministic timestep subdivision.
- **Packed bbox encoding**: `entityState_t.solid` encodes a symmetric bounding box in 24 bits (8 bits each for X half-width, Z-down, Z-up offset minus 32). This is a wire-compression technique — entity shapes are approximated to axis-aligned boxes for network transmission; bmodel entities skip this with a sentinel value (`SOLID_BMODEL`).
- **Command replay vs. extrapolation**: Modern engines (Valve's Source, Unreal) may extrapolate or use rollback. Q3's approach (full replay of all unacked commands) is simpler to implement correctly but more expensive. The `CMD_BACKUP` ring size sets the maximum latency before position freezes.
- **Predictable events**: `BG_AddPredictableEventToPlayerstate` stamps events with a sequence number; when the server confirms the same event, cgame suppresses the duplicate. This is an early form of what modern engines call "client-side prediction reconciliation" for non-physics state.
- **No scene graph**: Unlike modern ECS or scene-graph engines, collision queries iterate flat arrays with no spatial acceleration beyond the pre-filtered solid list. For the entity counts typical of Quake III (~64 entities), this was sufficient.

## Potential Issues

- **Prediction freeze under packet loss**: If the oldest cmd in the ring buffer is newer than `cg.snap->ps.commandTime` and also past `cg.time`, the function silently returns, holding the last valid predicted position. This can feel like momentary freezing under sustained high loss.
- **`cg_pmove` shared state across frames**: The static `cg_pmove` struct persists between frames. Fields like `tracemask` and `noFootsteps` are re-initialized each call, but any field not explicitly reset could carry stale data — a potential source of subtle prediction bugs if `pmove_t` is extended.
- **`pmove_msec` cvar clamping inside prediction**: Clamping a cvar mid-frame (`trap_Cvar_Set`) is a side-effectful operation with undefined ordering relative to other subsystems reading the same cvar that frame.
