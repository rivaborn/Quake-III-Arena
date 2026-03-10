# code/game/be_aas.h

## File Purpose
Public header exposing the Area Awareness System (AAS) interface to the game-side AI layer. It defines travel flags, spatial query result types, and movement prediction structures that bot AI code uses to navigate and reason about the world.

## Core Responsibilities
- Define all `TFL_*` travel type flags used to filter/allow navigation reachabilities
- Declare `aas_trace_t` for AAS-space sweep tests
- Declare `aas_entityinfo_t` for per-entity state visible to bots
- Declare `aas_areainfo_t` for querying area spatial/content metadata
- Define `SE_*` stop-event flags for client movement prediction
- Declare `aas_clientmove_t` for movement simulation results
- Declare `aas_altroutegoal_t` / `aas_predictroute_t` for alternate-route and route-prediction queries

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `solid_t` | enum | Collision solid type (none, trigger, bbox, BSP) |
| `aas_trace_t` | struct | Result of a swept-box trace through AAS space |
| `aas_entityinfo_t` | struct | Snapshot of an entity's state (position, animation, weapon, powerups) used by bots |
| `aas_areainfo_t` | struct | Spatial/content metadata for a single AAS area |
| `aas_clientmove_t` | struct | Output of client movement simulation (end position, velocity, stop event) |
| `aas_altroutegoal_t` | struct | An intermediate waypoint found by alternate-route search |
| `aas_predictroute_t` | struct | Output of route-ahead prediction (end pos, stop event, travel flags used) |

## Global / File-Static State
None.

## Key Functions / Methods
None — this is a pure header; no functions are defined here.

## Control Flow Notes
This header is included by game-side bot AI files (e.g., `ai_main.c`, `ai_dmq3.c`) to type-check calls into the botlib AAS API. The actual implementations live in `code/botlib/be_aas_*.c`. During each server frame, bot code calls AAS query functions (traces, route lookups, movement prediction) whose return types are the structs declared here.

## External Dependencies
- `qboolean`, `vec3_t` — defined in `q_shared.h` (engine shared types)
- `cplane_t` — referenced in the commented-out `bsp_trace_t` block; defined in `q_shared.h`
- `botlib.h` — noted inline as the canonical home for `bsp_trace_t` / `bsp_surface_t` (excluded via comment guard)
- `MAX_STRINGFIELD` — guarded define, may also be provided by botlib headers
