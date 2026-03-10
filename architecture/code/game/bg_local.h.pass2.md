# code/game/bg_local.h — Enhanced Analysis

## Architectural Role

This header serves as the **private integration layer for deterministic player movement shared between server and client VMs**. The "bg_" (both-game) module compiles `bg_pmove.c` and `bg_slidemove.c` identically into both the authoritative game VM (running on the server for snapshot generation) and the client-side cgame VM (running for client-side movement prediction). By declaring `pml_t` as a zeroed-every-frame scratch structure and exposing `pm` as a shared global pointer, this file enforces the architectural invariant that **the physics output is byte-identical regardless of execution context**, enabling the server to detect cheating via desync detection and allowing clients to smoothly predict unacknowledged movement.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/game/bg_pmove.c`** — Defines all the `extern` variables declared here and runs the main movement pipeline; the only translation unit that includes this header in the game VM
- **`code/game/bg_slidemove.c`** — Implements `PM_SlideMove` and `PM_StepSlideMove`, the core collision resolution routines
- **`code/cgame/cg_predict.c`** — Client-side prediction runs identical `Pmove` code; consumes the same `pml_t` and movement parameter globals
- **`code/game/g_active.c`** — Game VM's per-frame loop calls the public `Pmove()` entry point (not declared here but in `bg_public.h`); relies on `pml` state for frame-local computations
- **Movement parameters** (`pm_accelerate`, `pm_friction`, etc.) are **initialized in `code/game/bg_pmove.c`** but may be synchronized with server-side cvars (e.g., `pm_accelerate` tuning for balance patches)

### Outgoing (what this file depends on)

- **`code/game/bg_public.h`** — Defines `pmove_t` (the input/output movement structure) and the public `Pmove()` entry point; this header extends that contract with internal helpers
- **`q_shared.h` / `code/game/q_shared.c`** — Provides `vec3_t`, `trace_t`, `qboolean`, and foundational math (used by all movement code)
- **`code/qcommon/cm_*.c`** — Collision detection; `trace_t` results populated by `CM_Trace` are consumed by movement code (e.g., `groundTrace` in `pml_t` is the result of a collision test)

## Design Patterns & Rationale

**Deterministic Shared Physics via Code Duplication**: Rather than a network-synchronized state machine, Quake III achieves deterministic client prediction by compiling the identical movement code into both VM instances. The server runs `Pmove` authoritatively; the client re-runs it speculatively on unacknowledged commands. Any divergence (due to floating-point errors, cvar desync, or cheating) is detectable post-hoc. This was a novel approach in 1999—modern engines (e.g., Source 2, Unreal 5) use similar replay-based prediction.

**Per-Frame Zeroing**: `pml_t` is explicitly zeroed at the start of each movement frame because it holds accumulated derivatives (e.g., `impactSpeed`, `previous_origin`). This design prevents frame-to-frame state leakage and makes the physics stateless except for the input `pm->ps` structure—a key property for determinism.

**Extern Globals for Tuning**: Movement parameters are `extern float`s (defined in `bg_pmove.c`) rather than `#define` constants. This allows runtime cvar binding (e.g., `Cvar_Get("pm_accelerate", ...)`) and enables balance patches without recompilation—critical for a competitive multiplayer game where server admins tune physics.

## Data Flow Through This File

1. **Input**: Caller (game VM or cgame) populates `pmove_t *pm` with player state, velocity, input flags (jump, crouch, forward/strafe), and ground/water traces.
2. **Per-Frame Setup**: `pml_t` is **zeroed**, then populated with derived data:
   - Forward/right/up vectors computed from player angles
   - `groundPlane` and `groundTrace` filled from collision tests
   - `frametime` and `msec` set from clock
3. **Movement Resolution**:
   - `PM_StepSlideMove()` calls `PM_SlideMove()` (iterate up to 4 times, clipping against planes with `PM_ClipVelocity`)
   - Each clip updates `pm->ps.velocity` and `pm->ps.origin`
   - `PM_AddTouchEnt()` records collided entities for event generation; `PM_AddEvent()` queues footstep/land/jump sounds
4. **Output**: Modified `pm->ps` (origin, velocity, groundEntityNum) is returned to caller for snapshot/prediction use.

## Learning Notes

**Idiomatic to this engine/era**: 
- No object-oriented entity system; global per-frame scratch state (`pml_t`) is simpler than modern ECS component patterns but less composable.
- Physics constants as extern floats reflect the pre-data-driven era; a modern engine would load these from config files or database.
- The `OVERCLIP = 1.001` trick (slightly exceeding unit to escape numerical epsilon issues) is a subtle artifact of fixed-point physics—floating-point engines often use more robust epsilon handling.

**Contrasts with modern engines**:
- **No delta time in movement**: `frametime` and `msec` are fixed per-frame (typically 1/60 or 1 ms tick), not delta-time-aware. This simplifies determinism but couples the engine to a fixed tick rate.
- **Direct velocity clipping vs. impulses**: Rather than applying impulses and integrating velocity, the code directly modifies velocity and position, which is more efficient but less flexible for animation blending.
- **No animation-driven movement**: Movement is purely physics-based; modern engines blend animation and physics for smoother-looking foot placement.

**Why this matters for studying the engine**: This file is the *single most critical* for understanding Quake III's competitive robustness. The fact that client and server run identical code, with state zeroed each frame, is the foundation of its lag compensation and anti-cheat design. Learning to recognize this pattern (shared stateless simulation + replay verification) is essential for modern multiplayer game architecture.

## Potential Issues

- **Floating-point divergence**: While the code is compiled identically, floating-point operations are not bit-identical across platforms. A server running on Linux and a client on Windows could theoretically diverge. Quake III uses `Q_ftol()` (fast float-to-int) and careful rounding to minimize this, but some desync is possible on extreme jumps. Modern engines use fixed-point or per-frame reconciliation.
- **No explicit state validation**: There is no checksum or hash of `pml_t` to detect tampering. Cheating is detected post-hoc by comparing client-reported snapshots to server state, not by validating the physics state itself.
