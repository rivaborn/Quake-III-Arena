# code/cgame/cg_snapshot.c

## File Purpose
Manages the client-side snapshot pipeline, advancing the simulation clock by transitioning between server-delivered game state snapshots. It handles initial snapshot setup, interpolation state tracking, entity transitions, and teleport detection — all without necessarily firing every rendered frame.

## Core Responsibilities
- Read new snapshots from the client system into a double-buffered slot
- Initialize all entity state on the very first snapshot (or map restart)
- Transition `cg.nextSnap` → `cg.snap` when simulation time crosses the boundary
- Set `centity_t.interpolate` flags so the renderer knows whether to lerp or snap entities
- Detect teleport events (both entity-level and playerstate-level) and suppress interpolation accordingly
- Fire entity and playerstate events during snapshot transitions
- Record lagometer data for dropped/received snapshots

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `snapshot_t` | struct (defined elsewhere) | Server-delivered world state at a point in time |
| `centity_t` | struct | Per-entity client state including current/next `entityState_t` and interpolation flags |
| `cg_t` | struct | Global cgame frame state; holds `snap`, `nextSnap`, `activeSnapshots[2]`, timing |
| `cgs_t` | struct | Static cgame state; holds `processedSnapshotNum` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cg` | `cg_t` | global (extern) | Primary cgame state; `snap`/`nextSnap` pointers updated here |
| `cgs` | `cgs_t` | global (extern) | `processedSnapshotNum` incremented in `CG_ReadNextSnapshot` |
| `cg_entities` | `centity_t[MAX_GENTITIES]` | global (extern) | Per-entity client state written during all transition functions |

## Key Functions / Methods

### CG_ResetEntity
- **Signature:** `static void CG_ResetEntity( centity_t *cent )`
- **Purpose:** Hard-snaps an entity to its current state, bypassing interpolation; used when an entity first appears or teleports.
- **Inputs:** `cent` — the entity to reset
- **Outputs/Return:** void
- **Side effects:** Writes `cent->trailTime`, `cent->lerpOrigin`, `cent->lerpAngles`; calls `CG_ResetPlayerEntity` for player entities.
- **Calls:** `VectorCopy`, `CG_ResetPlayerEntity`
- **Notes:** Clears `previousEvent` only if the entity was absent longer than `EVENT_VALID_MSEC`.

---

### CG_TransitionEntity
- **Signature:** `static void CG_TransitionEntity( centity_t *cent )`
- **Purpose:** Promotes `cent->nextState` to `cent->currentState` and fires any pending events.
- **Inputs:** `cent`
- **Outputs/Return:** void
- **Side effects:** Modifies `cent->currentState`, `cent->currentValid`, `cent->interpolate`; may trigger sound/visual events via `CG_CheckEvents`.
- **Calls:** `CG_ResetEntity`, `CG_CheckEvents`
- **Notes:** Calls `CG_ResetEntity` when `cent->interpolate` is false (no prior valid frame or teleport).

---

### CG_SetInitialSnapshot
- **Signature:** `void CG_SetInitialSnapshot( snapshot_t *snap )`
- **Purpose:** Bootstraps the entire entity set on the first snapshot or after a tournament restart.
- **Inputs:** `snap` — the initial snapshot
- **Outputs/Return:** void
- **Side effects:** Sets `cg.snap`; writes all `cg_entities[]` currentState; calls `CG_BuildSolidList`, `CG_ExecuteNewServerCommands`, `CG_Respawn`, `CG_ResetEntity`, `CG_CheckEvents`.
- **Calls:** `BG_PlayerStateToEntityState`, `CG_BuildSolidList`, `CG_ExecuteNewServerCommands`, `CG_Respawn`, `memcpy`, `CG_ResetEntity`, `CG_CheckEvents`

---

### CG_TransitionSnapshot
- **Signature:** `static void CG_TransitionSnapshot( void )`
- **Purpose:** Advances `cg.snap` to `cg.nextSnap`, invalidating entities not in the new frame and transitioning those that are.
- **Inputs:** none (reads `cg.snap`, `cg.nextSnap`)
- **Outputs/Return:** void
- **Side effects:** Swaps snapshot pointers; updates all `cg_entities[]`; sets `cg.thisFrameTeleport`; may call `CG_TransitionPlayerState`.
- **Calls:** `CG_ExecuteNewServerCommands`, `BG_PlayerStateToEntityState`, `CG_TransitionEntity`, `CG_TransitionPlayerState`
- **Notes:** Dead-code branch `if ( !cg.snap ) {}` is a no-op left from map_restart handling.

---

### CG_SetNextSnap
- **Signature:** `static void CG_SetNextSnap( snapshot_t *snap )`
- **Purpose:** Stores an incoming snapshot as `cg.nextSnap` and sets per-entity `interpolate` flags.
- **Inputs:** `snap`
- **Outputs/Return:** void
- **Side effects:** Writes `cg.nextSnap`, entity `nextState` and `interpolate`; sets `cg.nextFrameTeleport`; calls `CG_BuildSolidList`.
- **Calls:** `BG_PlayerStateToEntityState`, `memcpy`, `CG_BuildSolidList`

---

### CG_ReadNextSnapshot
- **Signature:** `static snapshot_t *CG_ReadNextSnapshot( void )`
- **Purpose:** Pulls the next unprocessed snapshot from the client system into one of the two active slots.
- **Inputs:** none
- **Outputs/Return:** Pointer to a populated `snapshot_t`, or `NULL` if none available.
- **Side effects:** Increments `cgs.processedSnapshotNum`; calls `trap_GetSnapshot`; records lagometer data.
- **Calls:** `trap_GetSnapshot`, `CG_AddLagometerSnapshotInfo`, `CG_Printf`

---

### CG_ProcessSnapshots
- **Signature:** `void CG_ProcessSnapshots( void )`
- **Purpose:** Main per-frame entry point; drives the entire snapshot pipeline to keep `cg.snap`/`cg.nextSnap` correctly bracketing `cg.time`.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** May update `cg.snap`, `cg.nextSnap`, `cg.latestSnapshotNum`, `cg.time`; calls all subordinate snapshot functions.
- **Calls:** `trap_GetCurrentSnapshotNumber`, `CG_ReadNextSnapshot`, `CG_SetInitialSnapshot`, `CG_SetNextSnap`, `CG_TransitionSnapshot`, `CG_Error`
- **Notes:** Exits early if no snapshot has arrived yet. After the loop, clamps `cg.time` to `cg.snap->serverTime` to handle `vid_restart` edge case.

## Control Flow Notes

`CG_ProcessSnapshots` is called once per rendered frame (from `CG_DrawActiveFrame`), before prediction and rendering. It runs a `do-while` loop that keeps pulling and transitioning snapshots until `cg.time` sits in the interpolation window `[snap->serverTime, nextSnap->serverTime)`. Entity and player events are fired as a side-effect of transitions, not during rendering.

## External Dependencies

- **Includes:** `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere:**
  - `trap_GetCurrentSnapshotNumber`, `trap_GetSnapshot` — client system traps
  - `BG_PlayerStateToEntityState` — `bg_misc.c`
  - `CG_BuildSolidList` — `cg_predict.c`
  - `CG_ExecuteNewServerCommands` — `cg_servercmds.c`
  - `CG_Respawn`, `CG_TransitionPlayerState` — `cg_playerstate.c`
  - `CG_CheckEvents` — `cg_events.c`
  - `CG_ResetPlayerEntity` — `cg_players.c`
  - `CG_AddLagometerSnapshotInfo` — `cg_draw.c`
  - `cg`, `cgs`, `cg_entities` — `cg_main.c`
