# code/game/bg_local.h

## File Purpose
Internal header for the "bg" (both-game) player movement subsystem, shared between the game server and client-side prediction code. It declares the private `pml_t` locals struct, physics tuning constants, and exposes internal pmove helper function signatures that are used across the `bg_pmove.c` and `bg_slidemove.c` translation units.

## Core Responsibilities
- Define movement physics constants (slope limits, step height, jump velocity, timers)
- Declare `pml_t`, the per-frame local movement state that is zeroed before every `Pmove` call
- Expose `pm` and `pml` as extern globals shared across bg source files
- Declare extern movement parameter floats (speed, acceleration, friction tuning values)
- Expose the internal utility function prototypes used only within the bg subsystem

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `pml_t` | struct (typedef) | Per-frame local pmove state: player axis vectors, frame time, ground trace, impact speed, and previous-frame origin/velocity/waterlevel. Zeroed at the start of every pmove tick. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `pm` | `pmove_t *` | global (extern) | Pointer to the current pmove input/output structure passed in by the caller |
| `pml` | `pml_t` | global (extern) | Current frame's local movement scratch state |
| `pm_stopspeed` | `float` | global (extern) | Speed threshold at which the player is considered stopped |
| `pm_duckScale` | `float` | global (extern) | Velocity scale when crouching |
| `pm_swimScale` | `float` | global (extern) | Velocity scale when swimming |
| `pm_wadeScale` | `float` | global (extern) | Velocity scale when wading |
| `pm_accelerate` | `float` | global (extern) | Ground acceleration rate |
| `pm_airaccelerate` | `float` | global (extern) | Air acceleration rate |
| `pm_wateraccelerate` | `float` | global (extern) | Water acceleration rate |
| `pm_flyaccelerate` | `float` | global (extern) | Fly/noclip acceleration rate |
| `pm_friction` | `float` | global (extern) | Ground friction coefficient |
| `pm_waterfriction` | `float` | global (extern) | Water friction coefficient |
| `pm_flightfriction` | `float` | global (extern) | Flight friction coefficient |
| `c_pmove` | `int` | global (extern) | Debug/stat counter incremented each pmove invocation |

## Key Functions / Methods

### PM_ClipVelocity
- **Signature:** `void PM_ClipVelocity( vec3_t in, vec3_t normal, vec3_t out, float overbounce )`
- **Purpose:** Projects velocity `in` away from a surface plane defined by `normal`, writing the clipped result to `out`. Used to prevent the player from penetrating or sticking to geometry.
- **Inputs:** Incoming velocity, surface normal, overbounce factor (typically `OVERCLIP = 1.001`)
- **Outputs/Return:** Modified velocity written to `out`
- **Side effects:** None
- **Calls:** Not inferable from this file
- **Notes:** `OVERCLIP` slightly exceeds 1.0 to ensure the player stays clear of the plane numerically

### PM_AddTouchEnt
- **Signature:** `void PM_AddTouchEnt( int entityNum )`
- **Purpose:** Records an entity number in the pmove touch list so the game can process player-entity contact events after movement resolves.
- **Inputs:** Entity number of the touched entity
- **Outputs/Return:** void
- **Side effects:** Appends to `pm->touchents[]`
- **Calls:** Not inferable from this file

### PM_AddEvent
- **Signature:** `void PM_AddEvent( int newEvent )`
- **Purpose:** Queues a player event (e.g., land, jump, footstep) into the pmove event buffer for transmission to the client or game logic.
- **Inputs:** Event constant integer
- **Outputs/Return:** void
- **Side effects:** Modifies `pm->ps` event fields
- **Calls:** Not inferable from this file

### PM_SlideMove
- **Signature:** `qboolean PM_SlideMove( qboolean gravity )`
- **Purpose:** Iterative collision-slide movement: moves the player, clips against hit surfaces, and repeats up to a fixed number of planes to resolve complex corner cases.
- **Inputs:** `gravity` — whether to apply gravity during the slide
- **Outputs/Return:** `qtrue` if the player was blocked on any plane
- **Side effects:** Modifies `pm->ps.origin`, `pm->ps.velocity`, calls `PM_ClipVelocity`, `PM_AddTouchEnt`
- **Calls:** Not inferable from this file

### PM_StepSlideMove
- **Signature:** `void PM_StepSlideMove( qboolean gravity )`
- **Purpose:** Wraps `PM_SlideMove` with step-up logic: if the slide is blocked, attempts to step over the obstacle up to `STEPSIZE` (18 units) and re-run the slide.
- **Inputs:** `gravity`
- **Outputs/Return:** void
- **Side effects:** May adjust `pm->ps.origin` upward; calls `PM_SlideMove`
- **Calls:** `PM_SlideMove`

## Control Flow Notes
This header is included only by `bg_pmove.c` and `bg_slidemove.c`. Each call to `Pmove()` (the public entry point) zeroes `pml`, populates it from `pm`, then drives the movement pipeline: ground detection → acceleration → `PM_StepSlideMove` → event generation. This code runs identically on server (authoritative) and client (prediction), which is the entire purpose of the `bg_` (both-game) module.

## External Dependencies
- **`q_shared.h` / `bg_public.h`** — `vec3_t`, `trace_t`, `qboolean`, `pmove_t` types (defined elsewhere)
- `pmove_t` — defined in `bg_public.h`
- `vec3_t`, `trace_t` — defined in `q_shared.h`
- All `extern` variables are **defined** in `bg_pmove.c`
