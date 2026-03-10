# code/cgame/cg_snapshot.c — Enhanced Analysis

## Architectural Role

This file is the **temporal synchronization core** of the cgame VM. It sits at the exact boundary between the client engine layer (which delivers raw server snapshots via `trap_*` calls) and the rest of cgame (which assumes a stable, interpolation-ready entity world). Every other cgame subsystem — prediction, entity rendering, event audio, HUD — depends on the invariant that `cg.snap` and `cg.nextSnap` correctly bracket `cg.time`. This file enforces that invariant once per frame, before prediction or rendering begins. It is the single gate through which server-authoritative world state enters the client-visible simulation.

## Key Cross-References

### Incoming (who depends on this file)

- **`cg_view.c:CG_DrawActiveFrame`** — calls `CG_ProcessSnapshots()` as the first step of every rendered frame; nothing else in cgame runs until this returns with a valid `cg.snap`.
- **`cg_main.c`** — owns `cg`, `cgs`, and `cg_entities[]`; this file's writes to those globals are the primary side-effect consumed by all other cgame files.
- **`cg_predict.c`** — reads `cg.snap->ps`, `cg_entities[].currentState`, and `cg_entities[].interpolate` to run `Pmove` prediction. The `interpolate` flag set by `CG_SetNextSnap` determines whether prediction lerps or snaps.
- **`cg_ents.c`** — reads `cg_entities[].currentValid`, `.currentState`, `.nextState`, `.interpolate`, `.lerpOrigin`, `.lerpAngles` to position every packet entity. `CG_ResetEntity`'s writes to `lerpOrigin`/`lerpAngles` are the cold-start values for this pipeline.
- **`cg_players.c`** — `CG_ResetPlayerEntity` is called from `CG_ResetEntity`; the snapshot pipeline is the only trigger for player skeleton resets.

### Outgoing (what this file depends on)

- **Client engine trap layer** (`cg_syscalls.c` → `cl_cgame.c`): `trap_GetCurrentSnapshotNumber` and `trap_GetSnapshot` pull from the `clSnapshot_t` ring buffer maintained by `cl_parse.c`. This is the only inbound data source for the entire file.
- **`bg_misc.c:BG_PlayerStateToEntityState`** — called on every snapshot transition to keep the local player's `centity_t` in sync with the authoritative `playerState_t`. This is a shared-layer function compiled identically into game and cgame VMs.
- **`cg_predict.c:CG_BuildSolidList`** — called after both `CG_SetInitialSnapshot` and `CG_SetNextSnap` to rebuild the list of solid entities used for client-side collision prediction.
- **`cg_servercmds.c:CG_ExecuteNewServerCommands`** — called before entity transitions to flush any pending server string commands (configstring changes, scores, etc.) that arrived in the same snapshot.
- **`cg_playerstate.c`**: `CG_Respawn` (on initial snapshot) and `CG_TransitionPlayerState` (on non-predicted transitions — demo playback, spectating).
- **`cg_events.c:CG_CheckEvents`** — called per-entity during transition to fire audio/visual events; this is the primary mechanism by which server-side `EV_*` events reach the client.
- **`cg_draw.c:CG_AddLagometerSnapshotInfo`** — records per-snapshot quality data (drop or receive) for the on-screen lagometer.

## Design Patterns & Rationale

**Double-buffer ping-pong (`activeSnapshots[2]`):** `CG_ReadNextSnapshot` alternates between two pre-allocated `snapshot_t` slots to avoid allocation overhead and ensure the current `cg.snap` pointer remains stable while a new snapshot is being decoded into the other slot. This is a classic lock-free double-buffer pattern — no mutex needed because both reads and writes happen on the same thread (the cgame VM tick).

**Dirty-flag entity invalidation:** Before transitioning, all entities in the old snapshot have `currentValid = qfalse`. Only entities present in the new snapshot are re-validated by `CG_TransitionEntity`. Entities that disappear simply become invisible to `cg_ents.c` without any explicit removal step. This is efficient but means "entity left the PVS" and "entity was deleted" are indistinguishable to cgame.

**Event-as-state-transition side-effect:** Entity events (`EV_*` fields in `entityState_t`) are not polled; they fire exactly when `CG_TransitionEntity` calls `CG_CheckEvents`. This guarantees events fire exactly once per snapshot transition regardless of frame rate — a deliberate separation of simulation ticks from render ticks.

**Dead-code teleport guard:** The empty `if ( !cg.snap ) {}` block inside `CG_TransitionSnapshot` is a map_restart stub that was never implemented. It co-exists with correct post-swap behavior below it, suggesting the original plan was to reinitialize all entities on restart via this path but the implementation was abandoned in favor of using `CG_SetInitialSnapshot` directly.

## Data Flow Through This File

```
[Server UDP] → cl_parse.c (delta decompress) → clSnapshot_t ring buffer
                                                        │
                                            trap_GetSnapshot (cg_syscalls.c)
                                                        │
                                         CG_ReadNextSnapshot
                                         (ping-pong into activeSnapshots[2])
                                                        │
                              ┌─────────────────────────┴──────────────────────────┐
                              │ First frame                      │ All other frames  │
                    CG_SetInitialSnapshot               CG_SetNextSnap               │
                    (all entities hard-snapped)         (nextState written,          │
                              │                          interpolate flags set)      │
                              │                                   │                  │
                              └──────────── cg.time crosses boundary ───────────────┘
                                                        │
                                           CG_TransitionSnapshot
                                  (currentState ← nextState, events fired,
                                   playerState teleport detection,
                                   CG_TransitionPlayerState if non-predicted)
                                                        │
                                         cg.snap / cg_entities[] stable
                                                        │
                              ┌─────────────────────────┴────────────────────────────┐
                    cg_predict.c                  cg_ents.c                 cg_events.c
                  (Pmove on snap->ps)      (lerp entities to screen)   (audio/visuals)
```

## Learning Notes

**The snapshot model predates "interpolation" as an explicit design concept.** The `interpolate` flag is set per-entity based on whether the entity existed in both the old and new snapshots and did not teleport. This is the earliest widely-shipped implementation of what modern networking literature calls "entity interpolation" — the client always renders slightly behind the server clock (`cg.time` is typically `latency/2` ms behind), using two brackets to reconstruct smooth motion.

**`BG_PlayerStateToEntityState` as the bridge between two state representations:** Q3 maintains two parallel representations of the local player — `playerState_t` (the authoritative, prediction-friendly form) and `entityState_t` (the network-visible form). This file is the point where the snapshot's `ps` is converted to keep `cg_entities[clientNum]` consistent. This dual-representation pattern reflects the era's need to share entity arrays with the renderer while keeping prediction math clean.

**No explicit "snapshot ACK":** cgame never signals back to the client engine which snapshot it consumed. The client engine tracks this separately via sequence numbers and the `usercmd_t` acknowledgment system. `cg_snapshot.c` is purely a consumer.

**Modern contrast:** Modern engines (e.g., Source 2's entity system, Unreal's `UNetDriver`) separate snapshot processing from event dispatch into explicit phases and use explicit entity lifecycle callbacks. Q3's model conflates all of these into a single loop in `CG_TransitionSnapshot`, which is compact but makes it impossible to reorder or parallelize snapshot processing steps.

## Potential Issues

- **The `SNAPFLAG_NOT_ACTIVE` guard in `CG_ProcessSnapshots`** silently discards snapshots during level transitions without logging. If the transition takes longer than the snapshot queue depth (~32 entries in the client ring buffer), snapshots will be dropped permanently with no recovery path visible in this file.
- **Duplicate `serverTime` FIXME:** `CG_ReadNextSnapshot` has a commented-out `continue` for the case where a new snapshot has the same `serverTime` as `cg.snap`. The live code accepts the duplicate, which could cause `CG_TransitionSnapshot` to fire twice for the same logical server frame, potentially double-firing entity events.
- **`cg.time` floor clamp post-loop:** Clamping `cg.time = cg.snap->serverTime` on `vid_restart` silently erases any accumulated render-interpolation progress, which will cause a single-frame visual snap but is otherwise safe.
