# code/botlib/be_aas_debug.h

## File Purpose
Public header declaring AAS (Area Awareness System) debug visualization functions. Provides the interface for rendering temporary and permanent debug geometry (lines, crosses, polygons, arrows) and AAS data structures (faces, areas, reachabilities) into the game world.

## Core Responsibilities
- Declare functions for drawing temporary and permanent debug lines
- Declare functions for visualizing AAS primitives (faces, areas, reachabilities)
- Provide cross, arrow, and bounding-box rendering interfaces
- Expose polygon and plane-cross debug drawing
- Support travel-type diagnostic printing

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `aas_reachability_s` | struct (forward-declared) | Represents an AAS reachability link; used by `AAS_ShowReachability` |

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_ClearShownDebugLines
- Signature: `void AAS_ClearShownDebugLines(void)`
- Purpose: Clears all active temporary debug lines from the display.
- Inputs: None
- Outputs/Return: void
- Side effects: Modifies internal debug line list (defined in `be_aas_debug.c`)
- Calls: Not inferable from this file
- Notes: Paired with `AAS_ClearShownPolygons` for full debug state reset

### AAS_DebugLine
- Signature: `void AAS_DebugLine(vec3_t start, vec3_t end, int color)`
- Purpose: Renders a single transient debug line segment between two world-space points.
- Inputs: `start`, `end` — 3D endpoints; `color` — palette/index color
- Outputs/Return: void
- Side effects: Registers a line in the debug render list
- Calls: Not inferable from this file
- Notes: Non-permanent; cleared by `AAS_ClearShownDebugLines`

### AAS_PermanentLine
- Signature: `void AAS_PermanentLine(vec3_t start, vec3_t end, int color)`
- Purpose: Renders a line that persists across debug clear calls.
- Inputs: `start`, `end` — 3D endpoints; `color` — palette/index color
- Outputs/Return: void
- Side effects: Adds to permanent line list
- Calls: Not inferable from this file

### AAS_DrawPermanentCross / AAS_DrawCross
- Signature: `void AAS_DrawPermanentCross(vec3_t origin, float size, int color)` / `void AAS_DrawCross(vec3_t origin, float size, int color)`
- Purpose: Render a cross marker at a world-space origin; permanent vs. transient variants.
- Inputs: `origin` — world position; `size` — arm length; `color` — display color
- Outputs/Return: void
- Side effects: Adds geometry to debug render lists

### AAS_DrawPlaneCross
- Signature: `void AAS_DrawPlaneCross(vec3_t point, vec3_t normal, float dist, int type, int color)`
- Purpose: Draws a cross in the plane defined by a normal and distance, useful for visualizing BSP planes.
- Inputs: `point`, `normal`, `dist` — plane definition; `type`, `color` — display hints
- Outputs/Return: void
- Side effects: Adds debug geometry

### AAS_ShowBoundingBox
- Signature: `void AAS_ShowBoundingBox(vec3_t origin, vec3_t mins, vec3_t maxs)`
- Purpose: Renders an axis-aligned bounding box for debugging entity/area extents.
- Inputs: `origin`, `mins`, `maxs` — box definition
- Outputs/Return: void

### AAS_ShowFace / AAS_ShowArea / AAS_ShowAreaPolygons
- Signatures: `void AAS_ShowFace(int facenum)` / `void AAS_ShowArea(int areanum, int groundfacesonly)` / `void AAS_ShowAreaPolygons(int areanum, int color, int groundfacesonly)`
- Purpose: Visualize individual AAS faces or all faces within an area; optionally filter to ground-facing geometry only.
- Inputs: AAS face/area index integers; filter and color flags
- Side effects: Emits debug geometry via `AAS_DebugLine` or similar

### AAS_ShowReachability / AAS_ShowReachableAreas
- Signatures: `void AAS_ShowReachability(struct aas_reachability_s *reach)` / `void AAS_ShowReachableAreas(int areanum)`
- Purpose: Visualize a single reachability link or all links reachable from a given area (pathfinding debug).
- Inputs: Pointer to reachability struct or area index
- Side effects: Draws arrows/lines representing navigation graph edges

### AAS_DrawArrow
- Signature: `void AAS_DrawArrow(vec3_t start, vec3_t end, int linecolor, int arrowcolor)`
- Purpose: Draws a directed arrow for visualizing movement vectors or reachability directions.
- Inputs: Endpoints and separate colors for shaft and head

### AAS_PrintTravelType
- Signature: `void AAS_PrintTravelType(int traveltype)`
- Purpose: Prints a human-readable label for an AAS travel type constant to the debug output.
- Inputs: `traveltype` — AAS travel type integer constant
- Side effects: Console/log I/O

## Control Flow Notes
This is a pure header; no control flow here. The declared functions are called on-demand from AAS navigation code and bot AI code during development/debugging. They do not participate in the standard frame pipeline unless explicitly invoked by debug cvars or bot introspection routines.

## External Dependencies
- `vec3_t` — defined in `q_shared.h` (shared math types)
- `aas_reachability_s` — defined in `be_aas_reach.h` / `be_aas_def.h`
- Implementations reside in `code/botlib/be_aas_debug.c`
