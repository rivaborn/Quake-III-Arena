# code/cgame/cg_predict.c

## File Purpose
Generates `cg.predictedPlayerState` each frame by either interpolating between two server snapshots or running local client-side `Pmove` prediction on unacknowledged user commands. Also provides collision query utilities used by the prediction physics.

## Core Responsibilities
- Build a filtered sublist of solid and trigger entities from the current snapshot for efficient collision tests
- Provide `CG_Trace` and `CG_PointContents` wrappers that test against both world BSP and solid entities
- Interpolate player state between two snapshots when prediction is disabled or in demo playback
- Run client-side `Pmove` on all unacknowledged commands to predict the local player's position ahead of server acknowledgement
- Detect and decay prediction errors caused by server-vs-client divergence
- Predict item pickups and trigger interactions (jump pads, teleporters) locally

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `pmove_t` | struct (extern, defined in bg_public.h) | Input/output state passed to `Pmove`; holds player state, trace callbacks, command |
| `centity_t` | struct | Client entity with current/next state, lerp origin/angles |
| `playerState_t` | struct | Full player physics/game state snapshot |
| `snapshot_t` | struct | Server-delivered world state at a point in time |
| `trace_t` | struct | Result of a swept-box collision query |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cg_pmove` | `pmove_t` | static (file) | Persistent pmove context reused each prediction frame |
| `cg_numSolidEntities` | `int` | static (file) | Count of entities in `cg_solidEntities` |
| `cg_solidEntities` | `centity_t*[MAX_ENTITIES_IN_SNAPSHOT]` | static (file) | Filtered list of collidable entities for trace tests |
| `cg_numTriggerEntities` | `int` | static (file) | Count of entities in `cg_triggerEntities` |
| `cg_triggerEntities` | `centity_t*[MAX_ENTITIES_IN_SNAPSHOT]` | static (file) | Filtered list of trigger/item entities for touch tests |

## Key Functions / Methods

### CG_BuildSolidList
- **Signature:** `void CG_BuildSolidList( void )`
- **Purpose:** Partitions the active snapshot's entity list into solid collidables and trigger/item entities.
- **Inputs:** None (reads `cg.snap`, `cg.nextSnap`, `cg_entities[]`)
- **Outputs/Return:** Populates `cg_solidEntities`, `cg_triggerEntities`, and their counts.
- **Side effects:** Overwrites the four file-static arrays/counts.
- **Calls:** None (pure data classification).
- **Notes:** Prefers `cg.nextSnap` when no teleport is pending, so prediction traces use the most forward-in-time geometry.

### CG_ClipMoveToEntities
- **Signature:** `static void CG_ClipMoveToEntities( start, mins, maxs, end, skipNumber, mask, tr )`
- **Purpose:** Sweeps a box through all `cg_solidEntities`, updating `tr` with the nearest hit.
- **Inputs:** Ray endpoints, box extents, entity number to skip, content mask, in/out `trace_t`.
- **Outputs/Return:** Modifies `*tr` in place.
- **Side effects:** None beyond `*tr`.
- **Calls:** `trap_CM_InlineModel`, `trap_CM_TempBoxModel`, `trap_CM_TransformedBoxTrace`, `BG_EvaluateTrajectory`.
- **Notes:** Decodes packed bbox from `entityState_t.solid`; bmodel entities use lerp angles and trajectory-evaluated origin.

### CG_Trace
- **Signature:** `void CG_Trace( result, start, mins, maxs, end, skipNumber, mask )`
- **Purpose:** Public collision trace combining world BSP and solid entity sweeps.
- **Inputs:** Ray definition, skip entity, content mask.
- **Outputs/Return:** `*result` filled with closest hit.
- **Side effects:** None.
- **Calls:** `trap_CM_BoxTrace`, `CG_ClipMoveToEntities`.
- **Notes:** Assigned to `cg_pmove.trace` so `Pmove` uses client-side collision.

### CG_PointContents
- **Signature:** `int CG_PointContents( point, passEntityNum )`
- **Purpose:** Returns content flags at a world point, OR-ing in contributions from solid bmodel entities.
- **Inputs:** World point, entity to ignore.
- **Outputs/Return:** Combined content bitmask.
- **Side effects:** None.
- **Calls:** `trap_CM_PointContents`, `trap_CM_InlineModel`, `trap_CM_TransformedPointContents`.

### CG_InterpolatePlayerState
- **Signature:** `static void CG_InterpolatePlayerState( qboolean grabAngles )`
- **Purpose:** Lerps `cg.predictedPlayerState` between the two bracketing snapshots.
- **Inputs:** `grabAngles` — if true, overrides view angles from the latest user command instead of interpolating.
- **Outputs/Return:** Writes `cg.predictedPlayerState`.
- **Side effects:** May call `PM_UpdateViewAngles`.
- **Calls:** `trap_GetCurrentCmdNumber`, `trap_GetUserCmd`, `PM_UpdateViewAngles`, `LerpAngle`.
- **Notes:** Returns early without lerping if `cg.nextFrameTeleport` is set.

### CG_TouchItem
- **Signature:** `static void CG_TouchItem( centity_t *cent )`
- **Purpose:** Locally simulates picking up an item during prediction to keep weapon autoswitch and HUD responsive.
- **Inputs:** Candidate item entity.
- **Outputs/Return:** Modifies `cg.predictedPlayerState` and `cent->currentState`.
- **Side effects:** Sets `EF_NODRAW` on the item, stamps `cent->miscTime`, adds predictable event.
- **Calls:** `BG_PlayerTouchesItem`, `BG_CanItemBeGrabbed`, `BG_AddPredictableEventToPlayerstate`.
- **Notes:** Skips own-team flag pickup; guarded by `cg_predictItems` cvar.

### CG_TouchTriggerPrediction
- **Signature:** `static void CG_TouchTriggerPrediction( void )`
- **Purpose:** Tests the predicted player position against trigger entities to locally activate jump pads and mark teleporter hyperspace.
- **Inputs:** None (reads `cg_triggerEntities`, `cg.predictedPlayerState`).
- **Side effects:** Sets `cg.hyperspace`; calls `BG_TouchJumpPad`; may clear jumppad state.
- **Calls:** `CG_TouchItem`, `trap_CM_InlineModel`, `trap_CM_BoxTrace`, `BG_TouchJumpPad`.

### CG_PredictPlayerState
- **Signature:** `void CG_PredictPlayerState( void )`
- **Purpose:** Main per-frame entry point; drives the full prediction pipeline.
- **Inputs:** None (reads `cg.time`, `cg.snap`, `cg.nextSnap`, command buffer).
- **Outputs/Return:** Finalises `cg.predictedPlayerState`.
- **Side effects:** Calls `Pmove` (may mutate player state and fire events), updates `cg.predictedError`, calls `CG_TransitionPlayerState`.
- **Calls:** `CG_InterpolatePlayerState`, `Pmove`, `CG_TouchTriggerPrediction`, `CG_AdjustPositionForMover`, `CG_TransitionPlayerState`, `trap_GetCurrentCmdNumber`, `trap_GetUserCmd`, `trap_Cvar_Set`, `PM_UpdateViewAngles`, `CG_Printf`.
- **Notes:** Skips prediction entirely for demo playback or spectator-follow; decays prediction error via `cg_errorDecay` cvar; clamps `pmove_msec` to [8, 33].

## Control Flow Notes
- Called every render frame from `CG_DrawActiveFrame` (via `cg_view.c`), after `CG_ProcessSnapshots`.
- `CG_BuildSolidList` must be called whenever `cg.snap` changes (done in `cg_snapshot.c`) before any trace functions are valid.
- Prediction loop replays all commands from `current - CMD_BACKUP + 1` to `current`; commands already acknowledged (`serverTime <= commandTime`) are skipped.

## External Dependencies
- **Includes:** `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:**
  - `Pmove` — `bg_pmove.c`
  - `BG_EvaluateTrajectory`, `BG_PlayerTouchesItem`, `BG_CanItemBeGrabbed`, `BG_TouchJumpPad`, `BG_AddPredictableEventToPlayerstate`, `PM_UpdateViewAngles` — `bg_*.c`
  - `CG_AdjustPositionForMover`, `CG_TransitionPlayerState` — `cg_ents.c`, `cg_playerstate.c`
  - All `trap_CM_*` functions — cgame syscall layer (`cg_syscalls.c`)
  - `cg`, `cgs`, `cg_entities[]` — `cg_main.c`
