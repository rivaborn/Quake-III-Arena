# code/botlib/be_aas_debug.c

## File Purpose
Provides debug visualization utilities for the AAS (Area Awareness System) navigation data within the Quake III botlib. It renders debug lines and polygons representing AAS geometry (areas, faces, edges, reachabilities) via the engine's `botimport` interface.

## Core Responsibilities
- Manage a pool of reusable debug line handles (`debuglines[]`) and debug polygon handles (`debugpolygons[]`)
- Draw individual debug lines, permanent lines, crosses, arrows, and plane crosses into the world
- Visualize AAS faces as colored edge sequences or filled polygons
- Visualize AAS areas (all edges or ground-only polygons)
- Visualize `aas_reachability_t` records with directional arrows and movement prediction paths
- Cycle through and display all reachable areas from a given area over time
- Flood-fill and render all areas in a cluster starting from a world point

## Key Types / Data Structures
None defined locally; uses types from included headers.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `debuglines` | `int[1024]` | global | Engine-side debug line handles, indexed 0–1023 |
| `debuglinevisible` | `int[1024]` | global | Tracks which debug line slots are currently shown |
| `numdebuglines` | `int` | global | Count of allocated debug line slots |
| `debugpolygons` | `static int[8192]` | file-static | Engine-side debug polygon handles |

## Key Functions / Methods

### AAS_ClearShownPolygons
- Signature: `void AAS_ClearShownPolygons(void)`
- Purpose: Deletes all active debug polygons and zeros their handles.
- Inputs: None
- Outputs/Return: void
- Side effects: Calls `botimport.DebugPolygonDelete` for each non-zero slot; zeroes `debugpolygons[]`.
- Calls: `botimport.DebugPolygonDelete`

### AAS_ShowPolygon
- Signature: `void AAS_ShowPolygon(int color, int numpoints, vec3_t *points)`
- Purpose: Allocates a free polygon slot and creates a debug polygon.
- Inputs: color index, vertex count, vertex array
- Outputs/Return: void
- Side effects: Writes to `debugpolygons[]`; calls `botimport.DebugPolygonCreate`.

### AAS_ClearShownDebugLines
- Signature: `void AAS_ClearShownDebugLines(void)`
- Purpose: Deletes all active debug lines and resets state arrays.
- Inputs: None
- Side effects: Calls `botimport.DebugLineDelete`; zeroes `debuglines[]` and `debuglinevisible[]`.

### AAS_DebugLine
- Signature: `void AAS_DebugLine(vec3_t start, vec3_t end, int color)`
- Purpose: Finds a free or invisible debug line slot and shows a colored line segment.
- Inputs: start/end world positions, color index
- Side effects: Allocates new line handles via `botimport.DebugLineCreate` as needed; updates `debuglines[]`, `debuglinevisible[]`, `numdebuglines`.
- Notes: Re-uses existing invisible slots before allocating new ones.

### AAS_PermanentLine
- Signature: `void AAS_PermanentLine(vec3_t start, vec3_t end, int color)`
- Purpose: Creates a debug line outside the pool (not tracked, never reused).
- Side effects: Leaks handle — no slot stored; always calls `DebugLineCreate` + `DebugLineShow`.

### AAS_ShowFace
- Signature: `void AAS_ShowFace(int facenum)`
- Purpose: Draws all edges of an AAS face with cycling colors, plus a normal vector line.
- Inputs: face index into `aasworld.faces`
- Side effects: Calls `AAS_DebugLine` per edge and for normal visualization; calls `botimport.Print` on range error.
- Calls: `AAS_DebugLine`, `botimport.Print`, `VectorCopy`, `VectorMA`

### AAS_ShowArea
- Signature: `void AAS_ShowArea(int areanum, int groundfacesonly)`
- Purpose: Collects unique edge indices from an area and renders them as debug lines.
- Inputs: area number, flag to limit to ground/ladder faces
- Notes: Deduplicates edges in a local `areaedges[MAX_DEBUGLINES]` array before drawing.

### AAS_ShowReachability
- Signature: `void AAS_ShowReachability(aas_reachability_t *reach)`
- Purpose: Visualizes a single reachability link — draws destination area polygons, a directional arrow, and simulates the movement path for jump/rocket-jump/jumppad travel types.
- Calls: `AAS_ShowAreaPolygons`, `AAS_DrawArrow`, `AAS_HorizontalVelocityForJump`, `AAS_PredictClientMovement`, `AAS_RocketJumpZVelocity`, `AAS_JumpReachRunStart`, `AAS_DrawCross`

### AAS_ShowReachableAreas
- Signature: `void AAS_ShowReachableAreas(int areanum)`
- Purpose: Cycles through all reachabilities of an area, showing one every 1.5 seconds.
- Side effects: Uses `static` locals (`reach`, `index`, `lastareanum`, `lasttime`) to persist state across calls; calls `AAS_Time()`, `botimport.Print`.

### AAS_FloodAreas_r / AAS_FloodAreas
- Purpose: Recursively flood-fills and renders all areas in the same cluster as the given origin point, respecting view-portal and cluster boundaries.
- Side effects: `AAS_FloodAreas` allocates a zeroed `done[]` array via `GetClearedMemory`; `AAS_FloodAreas_r` calls `AAS_ShowAreaPolygons` per visited area.
- Notes: No `FreeMemory` call on the `done` array — potential memory leak.

## Control Flow Notes
This file is a pure debug/visualization utility. It has no init, frame, or shutdown hooks of its own. Functions are called on demand from higher-level bot debug commands or developer tools. `AAS_ShowReachableAreas` relies on being called repeatedly each frame to advance its time-based cycle via the static state variables.

## External Dependencies
- **botimport** (`be_interface.h`): `DebugLineCreate`, `DebugLineDelete`, `DebugLineShow`, `DebugPolygonCreate`, `DebugPolygonDelete`, `Print` — all defined in the engine/server layer
- **aasworld** (`be_aas_def.h`): Global AAS world state (areas, faces, edges, planes, vertexes, reachability, areasettings, edgeindex, faceindex)
- **aassettings** (`be_aas_def.h`): Physics constants (`phys_jumpvel`)
- `AAS_Time`, `AAS_PointAreaNum`, `AAS_AreaCluster`, `AAS_HorizontalVelocityForJump`, `AAS_PredictClientMovement`, `AAS_RocketJumpZVelocity`, `AAS_JumpReachRunStart` — defined in other `be_aas_*.c` files
- `GetClearedMemory` (`l_memory.h`), `Com_Memcpy` (`q_shared.h`), vector macros (`q_shared.h`)
