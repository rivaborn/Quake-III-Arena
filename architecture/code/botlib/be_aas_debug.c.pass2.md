# code/botlib/be_aas_debug.c — Enhanced Analysis

## Architectural Role

This file is the **debug visualization layer of the botlib AAS subsystem**. It bridges the AAS internal data model (areas, faces, edges, reachabilities) to the engine's rendering pipeline via `botimport` callbacks. It serves as a developer tool for visually inspecting navigation geometry during bot AI development and troubleshooting—**not a runtime-critical component**, but essential for understanding AAS correctness and debugging reachability issues. Functions are called on-demand through debug commands initiated from the server or developer console, never from the hot bot AI or pathfinding loops.

## Key Cross-References

### Incoming (who calls this file)

From the cross-reference index and architecture context:
- **be_aas_main.c** / **be_interface.c**: Called indirectly via console debug commands (e.g., `bot_debug_show_area`, `bot_debug_show_reachability`)
- **be_ai_dmq3.c** / **ai_dmnet.c** (game-side bot logic): May invoke AAS debug functions at the game VM boundary via `trap_BotLib*` syscalls if debug mode is enabled
- **Server console** (sv_ccmds.c, sv_main.c): High-level entry point for developer `bot_*` debug commands that delegate to botlib's public interface

### Outgoing (what this file depends on)

**AAS world data structures** (`be_aas_def.h`):
- `aasworld.faces[]`, `aasworld.areas[]`, `aasworld.edges[]`, `aasworld.vertexes[]`, `aasworld.planes[]`
- `aasworld.edgeindex[]`, `aasworld.faceindex[]`, `aasworld.reachability[]`, `aasworld.areareachability[]`
- `aasworld.clusters[]`, `aasworld.portals[]` (used indirectly by AAS_FloodAreas)
- `aasworld.areasettings[]`, `aasworld.numareas`, `aasworld.numfaces`, etc.

**botimport interface** (be_interface.h):
- `botimport.DebugLineCreate()`, `DebugLineDelete()`, `DebugLineShow()`
- `botimport.DebugPolygonCreate()`, `DebugPolygonDelete()`
- `botimport.Print()` for error reporting

**Peer AAS functions** (other be_aas_*.c files):
- `AAS_Time()` (be_aas_main.h) — for time-based state cycling in AAS_ShowReachableAreas
- `AAS_PointAreaNum()` (be_aas_sample.h) — for flood-fill origin lookup
- `AAS_AreaCluster()` (be_aas_sample.h) — for cluster boundary checks
- `AAS_HorizontalVelocityForJump()`, `AAS_RocketJumpZVelocity()`, `AAS_PredictClientMovement()` (be_aas_move.h) — movement simulation for reachability visualization
- `AAS_JumpReachRunStart()` (be_aas_move.h) — for jump path prediction

**Memory and utilities**:
- `GetClearedMemory()` (l_memory.h) — temporary scratch allocation in AAS_FloodAreas
- `Com_Memcpy()`, vector macros (q_shared.h)

## Design Patterns & Rationale

### 1. **Pool-Based Debug Handle Reuse**
The file maintains two fixed-size pools (`debuglines[1024]`, `debugpolygons[8192]`) of engine-side handles, with parallel "visibility" tracking (`debuglinevisible[]`). Rather than immediately deleting and reallocating on each draw call:
- Lines are marked invisible instead of freed, allowing re-use in the next frame
- `AAS_DebugLine()` finds the first invisible slot before creating a new handle
- `AAS_ClearShownDebugLines()` performs mass deletion and reset

**Rationale**: Minimize allocation churn; many debug visualizations are transient (per-frame reachability cycles). However, this trades memory for simplicity—engines with pooled allocators might manage this more dynamically.

### 2. **Static State Cycling in AAS_ShowReachableAreas**
```c
static aas_reachability_t *reach;
static int index;
static int lastareanum;
static int lasttime;
```
Uses C `static` locals to persist a "cursor" across repeated calls, advancing every ~1500ms. **Rationale**: Allows the developer to step through reachabilities over time without explicit external state, suitable for an interactive debug tool called each frame.

### 3. **Hardcoded Color Rotation**
Functions like `AAS_ShowFace()` and `AAS_ShowArea()` cycle through a fixed color palette (`LINECOLOR_RED → GREEN → BLUE → YELLOW → RED`). **Rationale**: Provides visual separation of edges within a geometric structure without requiring a parameter; idiomatic to this era's rendering (fixed palette indices).

### 4. **Lazy Flood-Fill with Temporary Allocation**
`AAS_FloodAreas()` allocates a temporary `done[aasworld.numareas]` via `GetClearedMemory()` but **never frees it** (potential memory leak). **Rationale**: Likely intended for one-time visualization per debug session; the engine's hunk allocator may auto-reset on map load, but this is implicit and fragile.

## Data Flow Through This File

1. **Input**: Developer invokes a debug command (e.g., `show_area 42`), which calls a function like `AAS_ShowArea(42, 0)`.

2. **Transformation**: 
   - Function walks the AAS graph structure (areas → faces → edges → vertexes)
   - Accumulates geometry (edge endpoints, polygon vertices)
   - May simulate movement (AAS_ShowReachability simulates jump arcs using AAS_HorizontalVelocityForJump)

3. **Output**: 
   - Calls `botimport.DebugLineCreate/Show()` or `DebugPolygonCreate()` to queue rendering in the engine
   - Engine-side handles are stored in `debuglines[]` / `debugpolygons[]` and persisted until next clear

4. **Persistence**: Lines remain visible until `AAS_ClearShownDebugLines()` clears the pool or the render thread consumes and flushes them.

## Learning Notes

### For a Developer Studying Q3A Navigation
This file is an **excellent reference for the AAS data model layout**:
- **Faces** are the 2D convex polygons that bound areas (edges define face boundaries)
- **Edges** are 1D line segments with bidirectional vertex references
- **Areas** are 3D convex volumes bounded by faces; they reference faces via `faceindex[]`
- **Reachabilities** are directed links between areas, labeled with travel type (walk, jump, ladder, teleport, etc.)

### Idiomatic Patterns of Q3A Era
- **Fixed-size arrays** instead of dynamic lists (common in 1999–2005 console game engines)
- **Pooling with bitflags** (debuglinevisible tracks state, no linked-list chaining)
- **Static locals for frame state** (predates entity component systems and explicit state objects)
- **Unidirectional imports**: botlib never calls game/engine; engine always drives via callbacks

### Modern Alternatives
- ECS engines would use an `DebugVisualizationComponent` and a `DebugRenderer` system
- Leak detection (ASAN) or explicit scope-based allocators would prevent the AAS_FloodAreas memory leak
- Enums or config tables would replace hardcoded color cycles

## Potential Issues

1. **Memory Leak in AAS_FloodAreas**: 
   - `done = GetClearedMemory(aasworld.numareas * sizeof(int))` is never freed
   - If called repeatedly during a dev session, the hunk fills up
   - Severity: Low (debug-only), but affects long debugging sessions

2. **Pool Overflow Risk**: 
   - If a debug visualization requests > 1024 lines or > 8192 polygons, silently fails with no warning
   - `AAS_DebugLine()` returns void, so caller can't detect overflow
   - Severity: Low (debug tool), but can cause silent visualization loss

3. **Hardcoded Line Limits**: 
   - `MAX_DEBUGLINES` is fixed to 1024, matching `aasworld` constraints elsewhere
   - Not parameterized; changing it requires recompilation

4. **No Timestamping on Pool Entries**: 
   - Older visualizations are kept indefinitely; no LRU or age-based cleanup
   - Suitable for a single "show once" debug workflow, not a continuous streaming visualization

5. **Potential Integer Overflow in Edge Index Lookups**: 
   - `edgenum = abs(aasworld.edgeindex[...])` — the abs() suggests signed indices encode direction, but no bounds check after abs()
   - If an area or face has a corrupt edgeindex, dereferencing `aasworld.edges[edgenum]` can crash
   - Mitigated by the range checks (`if (edgenum >= aasworld.numedges)`), but ordering matters
