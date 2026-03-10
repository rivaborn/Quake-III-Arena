# code/game/g_mover.c — Enhanced Analysis

## Architectural Role

`g_mover.c` is the **authoritative movement controller** for all dynamic geometry in the game module VM. It bridges three critical subsystems: (1) server-side entity simulation driven by the main frame loop, (2) the shared `BG_EvaluateTrajectory` layer that enables client-side prediction consistency, and (3) the qcommon collision system (`trap_Trace`, `trap_LinkEntity`, `trap_EntitiesInBox`). Mover state transitions and push mechanics are entirely server-authoritative, but their trajectories are evaluated client-side using identical code, making movers a key synchronization point between client prediction and server ground truth.

## Key Cross-References

### Incoming (who depends on this file)
- **`G_RunMover` called by:** `G_RunThink` dispatch loop in main frame thinking (once per entity per frame)
- **Spawn functions called by:** `G_SpawnEntitiesFromString` during BSP entity parsing at map load
- **Callbacks invoked by game logic:**
  - `ent->blocked(ent, obstacle)` — custom mover handlers (e.g., `Blocked_Door`) set by spawn functions
  - `ent->reached(ent)` — called when `TR_LINEAR_STOP` finishes; used for train logic
- **Binary mover activation:** `Use_BinaryMover` triggered by triggers and player touches

### Outgoing (what this file depends on)
- **qcommon collision system:** `trap_Trace`, `trap_EntitiesInBox`, `trap_LinkEntity`, `trap_UnlinkEntity` from server's collision world
- **Shared movement code:** `BG_EvaluateTrajectory` evaluated identically in cgame for prediction; `RadiusFromBounds`, `AngleVectors` from math library
- **Game module services:** `G_Damage` (crushing), `G_ExplodeMissile` (proximity mine detonation in MISSIONPACK), `G_AddEvent` (sound events), `TeleportPlayer` (door spectator bypass)
- **Configstring broadcast:** `trap_SetConfigstring` via `trap_AdjustAreaPortalState` (portal visibility gating tied to door state)

## Design Patterns & Rationale

**Atomic Push with Rollback** — The `pushed[]` global stack enables all entities affected by a mover's movement to be moved atomically: if any push fails, *all* are reverted in reverse order (line ~434). This prevents riders from sliding off and maintains the group invariant. Pre-modern engines favored this approach over per-object rollback queues used in modern ECS; the trade-off is simplicity (single stack) versus scalability (fixed MAX_GENTITIES array).

**Binary State Machine for Movers** — Each mover has exactly four states (`MOVER_POS1`, `POS2`, `1TO2`, `2TO1`), not continuous movement. `SetMoverState` configures the trajectory once and lets `BG_EvaluateTrajectory` replay it identically on server and client. This design decouples state transitions from frame rate and ensures client prediction matches server authority without per-frame delta-sync.

**Deferred Initialization via Think Callbacks** — Spawn functions (e.g., `SP_func_door`) schedule their heaviest work (trigger volume spawning, train path lookup) via `think` callbacks that fire one frame later (line ~550, `FRAMETIME` delay). This avoids BSP entity parse ordering problems and spreads load.

**Team Slaves for Coordinated Movement** — Multi-part movers (e.g., double doors) are linked via `teamchain` pointers; only the captain calls `G_MoverTeam`, which atomically moves all slaves. The `MatchTeam` helper syncs reversed/delayed slaves by adjusting their `trTime`. This is simpler than per-entity trajectory scheduling.

## Data Flow Through This File

1. **Spawn Phase (map load):**
   - `G_SpawnEntitiesFromString` → `SP_func_*` spawn functions
   - Extract BSP model, speed, sounds, travel targets from entity string
   - Call `InitMover` (common setup: trajectory, callbacks, model2, sounds)
   - Defer trigger/train setup via `think = Think_*` with `nextthink = level.time + FRAMETIME`

2. **Run Phase (per-server-frame):**
   - `G_RunMover` checks if entity is team slave; skips if so
   - If not stationary (`TR_STATIONARY`), call `G_MoverTeam`
   - `G_MoverTeam`: evaluate current position via `BG_EvaluateTrajectory`, compute delta move/amove
   - Reset `pushed_p` stack pointer, loop through team chain, call `G_MoverPush` for each
   - `G_MoverPush`: unlink pusher, query all entities in swept volume, attempt individual pushes
   - On success: link pusher, update all pushed entities, check if reached endpoint
   - On failure: revert all pushed entities from `pushed[]` stack, call `ent->blocked()`

3. **State Transitions:**
   - `Use_BinaryMover` (triggered) → call `MatchTeam` to sync slaves, then `SetMoverState` with new state
   - `SetMoverState` updates trajectory type, base position, duration; calls `trap_LinkEntity`
   - Client-side prediction reads the same `entityState_t` pos/apos trajectory and evaluates it with identical `BG_EvaluateTrajectory`

## Learning Notes

**Deterministic Shared Movement:** The decision to use `BG_EvaluateTrajectory` for both server authority and client prediction is a core design strength. Because trajectory parameters are set once at state transition and never re-evaluated frame-to-frame, both sides naturally stay in sync. Modern engines often use continuous physics or step-wise delta updates, which require more careful synchronization.

**No Per-Frame Physics Step:** Movers do not integrate velocity/acceleration per frame like dynamic bodies; they pre-compute their path and replay it. This is appropriate for rigid geometry and level design intended to be predictable. It contrasts with player movement (handled by `bg_pmove.c` and client-side prediction).

**Team Mechanics as Coordination:** The team/slave pattern predates entity component systems. Modern engines might use a "mover group" component or explicit synchronization constraint; here, a single `teamchain` pointer and a `MatchTeam` helper suffice for the Q3 use case (doors in pairs, trains with multiple cars).

**Proximity Mine Interaction (MISSIONPACK ifdef):** The special case at line ~379 reveals late-game polish: proximity mines attached to movers must detonate if crushed, adding gameplay depth. The `#ifdef MISSIONPACK` guard shows how mod variants diverged from base Q3A.

## Potential Issues

- **Stack Overflow Risk:** `pushed[MAX_GENTITIES]` is global and reset per team-move. In pathological cases (many simultaneous movers pushing the same large group), `pushed_p` could exceed bounds; the check at line ~137 will `G_Error`, but recovery is hard.
- **Bobbing Movers Instant-Kill (line ~410):** `TR_SINE` bobbing movers ignore blocking and damage any obstacle with `MOD_CRUSH`. This is a game-logic hack to prevent the player getting stuck, but it's surprising behavior for a crushing trap.
- **Portal State Coupling:** Door state changes directly call `trap_AdjustAreaPortalState`, tying rendering PVS to game logic state. Misaligned doors and portal state can cause visibility glitches.
