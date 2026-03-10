# code/botlib/be_aas_route.h

## File Purpose
Public (and internal) interface header for the AAS (Area Awareness System) routing subsystem of the Quake III bot library. It declares functions for computing travel times, querying reachabilities, predicting routes, and managing routing caches between AAS areas.

## Core Responsibilities
- Declare internal routing lifecycle functions (init, free, cache write) behind `AASINTERN` guard
- Expose travel-flag query helpers to external callers
- Provide area reachability enumeration API
- Expose travel-time computation between areas and to goal areas
- Expose route prediction with configurable stop events
- Allow dynamic enabling/disabling of areas for routing

## Key Types / Data Structures
None defined in this file; forward-declares structs from other headers.

| Name | Kind | Purpose |
|------|------|---------|
| `aas_reachability_s` | struct (forward ref) | Holds data for a single reachability link between areas |
| `aas_predictroute_s` | struct (forward ref) | Output record for a predicted route segment |

## Global / File-Static State
None declared in this file.

## Key Functions / Methods

### AAS_InitRouting *(AASINTERN only)*
- **Signature:** `void AAS_InitRouting(void)`
- **Purpose:** Initializes all routing data structures and caches for the loaded AAS file.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Allocates routing cache memory; modifies global AAS routing state.
- **Calls:** Not inferable from this file.
- **Notes:** Only compiled when `AASINTERN` is defined (i.e., inside the botlib itself).

### AAS_FreeRoutingCaches *(AASINTERN only)*
- **Signature:** `void AAS_FreeRoutingCaches(void)`
- **Purpose:** Frees all dynamically allocated routing cache memory.
- **Side effects:** Deallocates global routing cache state.

### AAS_AreaTravelTime
- **Signature:** `unsigned short int AAS_AreaTravelTime(int areanum, vec3_t start, vec3_t end)`
- **Purpose:** Returns the travel time (in AAS time units) to traverse within a single area from `start` to `end`.
- **Inputs:** `areanum` — area index; `start`/`end` — 3D positions within the area.
- **Outputs/Return:** Travel time as `unsigned short int`; 0 likely indicates unreachable.
- **Side effects:** None expected.
- **Notes:** Declared twice (once inside `AASINTERN`, once outside) — appears intentional to expose both internally and externally.

### AAS_AreaTravelTimeToGoalArea
- **Signature:** `int AAS_AreaTravelTimeToGoalArea(int areanum, vec3_t origin, int goalareanum, int travelflags)`
- **Purpose:** Returns the precomputed or cached travel time from a source area/origin to a goal area, filtered by allowed travel flags.
- **Inputs:** Source area, origin position, goal area, bitmask of allowed `TFL_*` travel flags.
- **Outputs/Return:** Travel time in AAS units; 0 if unreachable.
- **Side effects:** May populate routing cache on first query.

### AAS_PredictRoute
- **Signature:** `int AAS_PredictRoute(struct aas_predictroute_s *route, int areanum, vec3_t origin, int goalareanum, int travelflags, int maxareas, int maxtime, int stopevent, int stopcontents, int stoptfl, int stopareanum)`
- **Purpose:** Simulates a route from the given origin toward a goal, halting early if a configured stop condition is met (e.g., entering a hazardous content type or area).
- **Inputs:** Output route struct pointer; source area/origin; goal area; travel flags; limits (`maxareas`, `maxtime`); stop conditions (`stopevent`, `stopcontents`, `stoptfl`, `stopareanum`).
- **Outputs/Return:** Non-zero if a stop event was triggered; route struct populated with prediction result.
- **Side effects:** None beyond populating `*route`.
- **Notes:** Used by bots to detect danger zones along a planned path before committing.

### AAS_EnableRoutingArea
- **Signature:** `int AAS_EnableRoutingArea(int areanum, int enable)`
- **Purpose:** Dynamically marks an area as enabled or disabled for routing, allowing the game to block bot navigation through specific regions at runtime.
- **Inputs:** `areanum` — target area; `enable` — non-zero to enable, 0 to disable.
- **Outputs/Return:** Previous enabled state or status code.
- **Side effects:** Modifies global routing area state; invalidates related caches.

### Notes on remaining functions
- `AAS_TravelFlagForType` / `AAS_AreaContentsTravelFlags` — convert travel type enums and area content flags into `TFL_*` bitmasks used to filter routing queries.
- `AAS_NextAreaReachability` / `AAS_ReachabilityFromNum` — iterator pair for enumerating all reachability links out of an area.
- `AAS_RandomGoalArea` — selects a random reachable goal area given travel constraints, used for bot wandering/exploration.

## Control Flow Notes
This header is included by both internal botlib modules (`AASINTERN` defined) and external callers such as `be_ai_move.c` and `be_ai_goal.c`. At map load, `AAS_InitRouting` is called once. Per-frame, bots call `AAS_AreaTravelTimeToGoalArea` and `AAS_PredictRoute` to select and validate movement paths. `AAS_EnableRoutingArea` may be called in response to game events (e.g., doors, triggers).

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `aas_reachability_s`, `aas_predictroute_s` — defined in `be_aas_def.h` or `aasfile.h`
- Travel flag constants (`TFL_*`) — defined in `be_aas_move.h` or `aasfile.h`
- Implementation: `be_aas_route.c`
