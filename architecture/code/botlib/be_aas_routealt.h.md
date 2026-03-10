# code/botlib/be_aas_routealt.h

## File Purpose
Public and internal interface header for the AAS (Area Awareness System) alternative routing subsystem. It exposes functions for computing alternative route goals between two AAS areas, used by the bot AI to find tactically varied paths.

## Core Responsibilities
- Declares internal (`AASINTERN`) lifecycle functions for the alternative routing system
- Exposes the public API for querying alternative route goals to bot clients
- Guards internal symbols behind the `AASINTERN` preprocessor gate, enforcing module encapsulation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_altroutegoal_t` | struct (defined elsewhere) | Represents a single alternative route waypoint/goal candidate |
| `vec3_t` | typedef (defined elsewhere) | 3-element float vector for world-space positions |

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_InitAlternativeRouting
- Signature: `void AAS_InitAlternativeRouting(void)`
- Purpose: Initializes internal state and data structures for the alternative routing system.
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates or resets module-level routing state (inferred; implementation not visible here)
- Calls: Not inferable from this file
- Notes: Only visible when `AASINTERN` is defined; called from botlib internal init sequence

### AAS_ShutdownAlternativeRouting
- Signature: `void AAS_ShutdownAlternativeRouting(void)`
- Purpose: Tears down and frees resources owned by the alternative routing subsystem.
- Inputs: None
- Outputs/Return: None
- Side effects: Frees allocated memory / resets internal state
- Calls: Not inferable from this file
- Notes: Paired with `AAS_InitAlternativeRouting`; internal-only

### AAS_AlternativeRouteGoals
- Signature: `int AAS_AlternativeRouteGoals(vec3_t start, int startareanum, vec3_t goal, int goalareanum, int travelflags, aas_altroutegoal_t *altroutegoals, int maxaltroutegoals, int type)`
- Purpose: Computes a set of alternative intermediate route goals between a start and goal position/area, allowing bots to take tactically distinct paths rather than always following the single shortest route.
- Inputs:
  - `start` — world-space origin of the bot
  - `startareanum` — AAS area index containing `start`
  - `goal` — world-space target position
  - `goalareanum` — AAS area index containing `goal`
  - `travelflags` — bitmask of allowed travel types (walk, swim, jump, etc.)
  - `altroutegoals` — caller-allocated output array for results
  - `maxaltroutegoals` — capacity of the output array
  - `type` — selects the alternative routing strategy/mode
- Outputs/Return: Number of alternative route goals written into `altroutegoals`; returns `int`
- Side effects: Writes into the caller-supplied `altroutegoals` buffer
- Calls: Not inferable from this file
- Notes: Public API; available to all botlib consumers without `AASINTERN`

## Control Flow Notes
- `AAS_InitAlternativeRouting` / `AAS_ShutdownAlternativeRouting` are called during botlib init and shutdown respectively (within the `AASINTERN`-compiled translation units).
- `AAS_AlternativeRouteGoals` is called per-frame or per-decision-cycle by bot AI goal selection logic to populate a candidate list of waypoints, enabling path diversity.

## External Dependencies
- `aas_altroutegoal_t` — defined in `aasfile.h` or `be_aas_def.h`
- `vec3_t` — defined in `q_shared.h`
- `AASINTERN` — preprocessor macro controlling visibility of internal symbols
- Implementation resides in `be_aas_routealt.c` (defined elsewhere)
