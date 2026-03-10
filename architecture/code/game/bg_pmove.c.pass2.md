# code/game/bg_pmove.c — Enhanced Analysis

## Architectural Role

`bg_pmove.c` is the single most strategically important file in the Q3A codebase for multiplayer correctness: it is compiled **identically** into both the game VM (`code/game/`) and the cgame VM (`code/cgame/`), making it the physical contract that enables client-side prediction. The server runs `Pmove` authoritatively in `g_active.c:ClientThink`; the client reruns the same code in `cg_predict.c` against buffered unacknowledged `usercmd_t`s, then decays any divergence from the server's authoritative `playerState_t`. Any non-determinism or platform difference in this file immediately manifests as prediction error and rubber-banding. It sits at the intersection of four subsystems: game VM, cgame VM, qcommon (collision via `pm->trace`), and the networking layer (whose delta-compressed `playerState_t` wire format is the direct output of this code).

## Key Cross-References

### Incoming (who depends on this file)

- **`code/game/g_active.c`** — calls `Pmove(pmove)` once per `ClientThink` tick; this is the authoritative server-side simulation that produces the canonical `playerState_t` broadcast in snapshots.
- **`code/cgame/cg_predict.c`** — calls `Pmove` in a replay loop over unacknowledged `usercmd_t`s to produce a predicted `playerState_t` used for local rendering (no network round-trip).
- **`code/game/bg_slidemove.c`** — directly called by this file (`PM_SlideMove`, `PM_StepSlideMove`); the two files form a single logical unit split for compilation size reasons. `bg_slidemove.c` also follows the dual-compilation contract.
- **`code/botlib/be_aas_move.c`** — implements its own parallel movement simulation (`AAS_PredictClientMovement`, `AAS_ApplyFriction`, `AAS_Accelerate`) that **mirrors** the physics constants defined here. Changes to `pm_friction`, `pm_accelerate`, etc. must be manually propagated to botlib or bot pathfinding will diverge from actual player movement.

### Outgoing (what this file depends on)

- **`code/game/bg_misc.c`** — `BG_AddPredictableEventToPlayerstate`; the "predictable event" system ensures both client and server generate identical event sequences in lockstep.
- **`code/game/bg_slidemove.c`** — `PM_StepSlideMove` / `PM_SlideMove`; recursive velocity clipping against multiple contact planes.
- **`trap_SnapVector`** (syscall boundary) — snaps float velocity components to integers before transmission; the VM syscall resolves to different native implementations per platform (`snapvector.nasm` on x86 Linux, `ftol.nasm`, etc. via `code/unix/`) but must produce identical results on server and client.
- **`pm->trace` / `pm->pointcontents`** — function pointers injected at call time; on the server these resolve to `SV_Trace`/`SV_PointContents` from `code/server/sv_world.c`; on the client (cgame) they resolve to `trap_CM_*` calls into the collision model from `code/qcommon/cm_trace.c`.
- **`q_shared.c` / `q_math.c`** — `AngleVectors`, `DotProduct`, `VectorNormalize`, `VectorMA`, `VectorLength`.

## Design Patterns & Rationale

- **Global context pointer (`pm`, `pml`)**: Rather than threading context through every static helper, a single `pm` pointer is set at `PmoveSingle` entry. This is safe because the Q3A game loop is strictly single-threaded per VM instance, and the botlib has its own separate simulation. It mirrors Quake 1/2's `pmove` globals, reflecting a deliberate carry-over rather than a design oversight.
- **Intentional bug preservation (`#if 0` acceleration block)**: The commented-out "proper" `PM_Accelerate` implementation (using `VectorSubtract`/`VectorNormalize`) would have eliminated the famous Q3 strafe-jump speed accumulation. It was deliberately left disabled, preserving strafe-jumping as a skill mechanic. The comment explicitly says "feels bad," revealing a gameplay-over-correctness tradeoff.
- **Toggle-bit animation encoding**: `legsAnim` and `torsoAnim` encode both the animation index and a 1-bit toggle (bit 10). Flipping the toggle on each new animation start forces cgame to restart the animation even when reusing the same index — a compact bandwidth-efficient notification mechanism that avoids a separate "animation changed" field in `playerState_t`.
- **Time subdivision in `Pmove`**: Capping sub-steps at 66ms (≈15Hz) decouples physics from server frame rate. A server running at 125Hz still produces the same trajectory as one at 20Hz, within sub-step granularity. This is the engine-level answer to the "faster CPU = faster movement" bug that plagued Quake 1.

## Data Flow Through This File

```
usercmd_t (from network / input)
    │
    ▼
Pmove() ──subdivides time──► PmoveSingle() [1–N times]
    │                            │
    │                     pm->trace() ──► sv_world/cm_trace [collision]
    │                            │
    │                     velocity transform pipeline:
    │                     PM_GroundTrace → PM_SetWaterLevel → PM_CheckDuck
    │                     → [PM_WalkMove | PM_AirMove | PM_WaterMove | PM_FlyMove | …]
    │                       └─ PM_Friction → PM_Accelerate → PM_StepSlideMove
    │                            │
    │                     PM_Weapon → ammo/weaponstate mutation
    │                            │
    │                     PM_Footsteps / PM_WaterEvents → BG_AddPredictableEvent
    │                            │
    │                     trap_SnapVector(velocity) ← bandwidth optimization
    ▼
playerState_t (mutated in place) ──► game: snapshot delta-compress → clients
                                 └──► cgame: local render / HUD
```

Predictable events flow into `playerState_t.events[]` (ring buffer) via `bg_misc.c:BG_AddPredictableEventToPlayerstate`, which is the synchronization point ensuring both server and client fire the same sounds/effects at the same moment.

## Learning Notes

- **Client-side prediction architecture**: This file is the canonical example of how to share physics code for prediction. The pattern—identical source compiled into both VMs, callback-injected collision, deterministic integer snapping—predates modern game engine ECS by a decade but solves the same latency-hiding problem. Modern engines (Source, UE4+) use similar "shared game code" or "network prediction" modules.
- **`bg_` prefix convention**: Files prefixed `bg_` ("both games") are the Q3A idiom for the shared-physics contract. Understanding this prefix immediately tells you a file must be deterministic, must not use platform-specific floating point, and must not call any engine service except through the `pm->` callback table and `trap_SnapVector`.
- **Powerup integration in physics**: `PW_FLIGHT`, `PW_HASTE` are queried directly in the movement and weapon code rather than being abstracted behind a capability flag. This couples item data to the physics system — a pattern modern engines avoid by separating movement modifiers.
- **Weapon state machine vs. game logic**: `PM_Weapon` does more than animation; it consumes ammo and generates `EV_FIRE_WEAPON`. This means the physics step drives shot timing, not a separate game-logic system. The server's authoritative `Pmove` call is therefore also authoritative for shot timing — a clean design that avoids a separate fire-rate replication channel.

## Potential Issues

- **botlib physics drift**: `be_aas_move.c` hard-codes its own copies of friction/acceleration constants rather than reading the globals defined here. Any tuning change to `pm_friction` or `pm_accelerate` will silently break bot jump-arc validation without producing a compile error.
- **Non-reentrant globals**: `pm` and `pml` are process-global. Running two simultaneous `Pmove` calls (e.g., in a hypothetical multi-threaded server) would corrupt both. Not a current issue, but worth noting for any multi-threaded port.
- **`trap_SnapVector` contract**: The correctness of prediction depends on `trap_SnapVector` producing bit-identical results on server and client. On platforms where the syscall resolves differently (e.g., x87 vs. SSE rounding modes), prediction errors are possible — a known source of platform-specific divergence in Q3 mods.
