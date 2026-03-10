# code/botlib/be_aas_funcs.h

## File Purpose
A convenience aggregation header for the AAS (Area Awareness System) subsystem of the Quake III botlib. It acts as a single-include facade that pulls in all AAS sub-module headers, conditional on not being compiled as part of the BSPC map compiler tool.

## Core Responsibilities
- Aggregates all AAS sub-module public (and internal) headers into one include
- Guards inclusion behind `#ifndef BSPCINCLUDE` to prevent use in the BSPC offline tool
- Provides a single include point for any translation unit needing full AAS API access

## Key Types / Data Structures
None — this file declares no types of its own; all types originate in the bundled headers.

## Global / File-Static State
None.

## Key Functions / Methods
None — this file declares no functions directly. All function declarations are delegated to the included headers:

| Included Header | Responsibility |
|---|---|
| `be_aas_main.h` | AAS lifecycle: init, shutdown, map load, frame start |
| `be_aas_entity.h` | Entity tracking, BSP link management, entity queries |
| `be_aas_sample.h` | Point/area queries, traces, bounding box area lookup |
| `be_aas_cluster.h` | Area cluster initialization and portal classification |
| `be_aas_reach.h` | Reachability computation and area property queries |
| `be_aas_route.h` | Routing/pathfinding queries |
| `be_aas_routealt.h` | Alternate route calculation |
| `be_aas_debug.h` | Debug visualization helpers |
| `be_aas_file.h` | AAS file I/O |
| `be_aas_optimize.h` | AAS data optimization pass |
| `be_aas_bsp.h` | BSP-to-AAS interface |
| `be_aas_move.h` | Bot movement prediction and query |

## Control Flow Notes
This header is not tied to any specific engine phase directly. It is included by botlib internal translation units (e.g., `be_interface.c`, AI modules) that need the complete AAS API surface. The `BSPCINCLUDE` guard ensures the BSPC map-compiler tool—which shares some botlib source files but uses its own AAS stub—does not pull in runtime AAS headers that would conflict with or be unnecessary for offline BSP compilation.

## External Dependencies
- All dependencies are local botlib headers listed above
- The `BSPCINCLUDE` macro is defined externally by the BSPC build system; its absence enables the includes
- The `AASINTERN` macro (used inside several bundled headers) gates internal-only declarations for botlib-internal translation units vs. external callers
