# code/botlib/aasfile.h
## File Purpose
Defines the binary file format for AAS (Area Awareness System) data used by the Quake III bot navigation system. It declares all constants, flags, and on-disk data structures that describe how navigable areas, connectivity (reachability), and spatial partitioning are stored in `.aas` files.

## Core Responsibilities
- Define the AAS file magic identifier and version constants
- Enumerate all travel types bots can use to move between areas
- Define face, area content, and area flag bitmasks
- Declare the 14 lump layout for the AAS binary file format
- Provide all geometric and topological structs (vertices, edges, faces, areas, nodes)
- Define cluster/portal structures for hierarchical routing
- Declare the file header struct that indexes all lumps

## External Dependencies
- No explicit includes in this header itself
- Depends on `vec3_t` being defined by the including translation unit (from `q_shared.h`)
- Structs are consumed by: `be_aas_file.c`, `be_aas_route.c`, `be_aas_move.c`, `be_aas_reach.c`, `be_aas_cluster.c`, and `bspc/` tool sources

# code/botlib/be_aas_bsp.h
## File Purpose
Header for the AAS (Area Awareness System) BSP interface within the botlib. It declares functions for BSP-space collision, visibility, entity querying, and BSP model metadata used by the bot navigation system.

## Core Responsibilities
- Declare internal (AASINTERN) BSP file load/dump and entity-link management functions
- Expose public trace and point-contents queries into the BSP world
- Provide PVS/PHS visibility tests between world points
- Expose area connectivity queries for AAS routing
- Support entity enumeration and key-value (epair) property lookup on BSP entities

## External Dependencies
- `bsp_link_t`, `bsp_trace_t`, `vec3_t`, `qboolean` — defined elsewhere (likely `be_aas_def.h` / `q_shared.h`)
- `#define MAX_EPAIRKEY 128` — only constant defined in this file
- Implementation in `be_aas_bspq3.c` (defined elsewhere)

# code/botlib/be_aas_bspq3.c
## File Purpose
Provides the BSP world interface layer for the Q3 bot library (botlib), bridging AAS (Area Awareness System) navigation code to the engine's collision/BSP subsystem via the `botimport` callback table. It also owns the BSP entity data store, parsing and exposing map entity key-value pairs to the rest of botlib.

## Core Responsibilities
- Delegate spatial queries (traces, point contents, PVS/PHS tests) to engine callbacks via `botimport`
- Load and cache BSP entity lump data from the engine into `bspworld`
- Parse the raw entity text into a queryable linked-list of `bsp_entity_t` / `bsp_epair_t` records
- Provide typed accessors for entity key-value pairs (string, vector, float, int)
- Provide entity iteration (`AAS_NextBSPEntity`) and range validation
- Stub out unused BSP spatial linking functions (`AAS_UnlinkFromBSPLeaves`, `AAS_BSPLinkEntity`, `AAS_BoxEntities`)

## External Dependencies
- `../game/q_shared.h` — shared types (`vec3_t`, `qboolean`, `Com_Memcpy`, `Com_Memset`)
- `l_memory.h` — `GetHunkMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_script.h` — script tokeniser (`LoadScriptMemory`, `PS_ReadToken`, `PS_ExpectTokenType`, `FreeScript`, etc.)
- `be_aas_def.h` / `be_aas_funcs.h` — AAS internal types (`bsp_trace_t`, `bsp_link_t`)
- `aasfile.h` — AAS file format constants
- `../game/botlib.h` — `botlib_import_t` definition, `BLERR_NOERROR`, `MAX_EPAIRKEY`
- `../game/be_aas.h` — public AAS interface types
- `botimport` — defined in `be_interface.c`; all engine calls go through this table

# code/botlib/be_aas_cluster.c
## File Purpose
Implements AAS (Area Awareness System) area clustering for the Quake III bot navigation library. It partitions the navigable world into clusters separated by portal areas, enabling efficient hierarchical pathfinding by reducing the search space within each cluster.

## Core Responsibilities
- Identify and create portal areas that act as boundaries between clusters
- Flood-fill cluster membership from seed areas using reachability links (and optionally face adjacency)
- Assign cluster-local area indices to all areas and portals within each cluster
- Detect and heuristically generate candidate portal areas (`AAS_FindPossiblePortals`)
- Validate that every portal has exactly one front and one back cluster
- Initialize and reinitialize the full clustering data structures (`AAS_InitClustering`)
- Manage view portals (a superset of cluster portals used for PVS)

## External Dependencies
- **Includes:** `q_shared.h`, `l_memory.h`, `l_script.h`, `l_precomp.h`, `l_struct.h`, `l_log.h`, `l_libvar.h`, `aasfile.h`, `botlib.h`, `be_aas.h`, `be_aas_funcs.h`, `be_aas_def.h`
- **Defined elsewhere:**
  - `aasworld` — global AAS world state (`be_aas_main.c`)
  - `botimport` — engine import struct (`be_interface.c`)
  - `AAS_AreaReachability` — reachability query (`be_aas_reach.c` / `be_aas_funcs.h`)
  - `AAS_Error`, `Log_Write` — error/log utilities (`be_aas_main.c`, `l_log.c`)
  - `GetClearedMemory`, `FreeMemory` — bot memory allocator (`l_memory.c`)
  - `LibVarGetValue` — library variable query (`l_libvar.c`)

# code/botlib/be_aas_cluster.h
## File Purpose
Header file for the AAS (Area Awareness System) clustering subsystem within the botlib. It declares internal functions for initializing area clusters and designating view portals as cluster portals, guarded behind the `AASINTERN` preprocessor gate.

## Core Responsibilities
- Declares the clustering initialization entry point for internal AAS use
- Declares the function to promote view portals to cluster portal status
- Guards all declarations behind `AASINTERN` so they are invisible to external callers

## External Dependencies
- No includes in this file.
- `AASINTERN` macro — defined externally (likely in `be_aas_def.h` or a compilation unit that includes internal AAS headers) to gate visibility of these declarations.
- Implementation defined in `code/botlib/be_aas_cluster.c`.

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

## External Dependencies
- **botimport** (`be_interface.h`): `DebugLineCreate`, `DebugLineDelete`, `DebugLineShow`, `DebugPolygonCreate`, `DebugPolygonDelete`, `Print` — all defined in the engine/server layer
- **aasworld** (`be_aas_def.h`): Global AAS world state (areas, faces, edges, planes, vertexes, reachability, areasettings, edgeindex, faceindex)
- **aassettings** (`be_aas_def.h`): Physics constants (`phys_jumpvel`)
- `AAS_Time`, `AAS_PointAreaNum`, `AAS_AreaCluster`, `AAS_HorizontalVelocityForJump`, `AAS_PredictClientMovement`, `AAS_RocketJumpZVelocity`, `AAS_JumpReachRunStart` — defined in other `be_aas_*.c` files
- `GetClearedMemory` (`l_memory.h`), `Com_Memcpy` (`q_shared.h`), vector macros (`q_shared.h`)

# code/botlib/be_aas_debug.h
## File Purpose
Public header declaring AAS (Area Awareness System) debug visualization functions. Provides the interface for rendering temporary and permanent debug geometry (lines, crosses, polygons, arrows) and AAS data structures (faces, areas, reachabilities) into the game world.

## Core Responsibilities
- Declare functions for drawing temporary and permanent debug lines
- Declare functions for visualizing AAS primitives (faces, areas, reachabilities)
- Provide cross, arrow, and bounding-box rendering interfaces
- Expose polygon and plane-cross debug drawing
- Support travel-type diagnostic printing

## External Dependencies
- `vec3_t` — defined in `q_shared.h` (shared math types)
- `aas_reachability_s` — defined in `be_aas_reach.h` / `be_aas_def.h`
- Implementations reside in `code/botlib/be_aas_debug.c`

# code/botlib/be_aas_def.h
## File Purpose
Central internal definition header for the AAS (Area Awareness System) botlib subsystem. Defines all major data structures, constants, and macros used by the AAS pathfinding and navigation system, and acts as the aggregating include for all AAS subsystem headers.

## Core Responsibilities
- Define the monolithic `aas_t` world state structure holding all AAS geometry, routing, and entity data
- Define entity link structures for spatial partitioning across AAS areas and BSP leaves
- Define routing cache and routing update structures for the pathfinding algorithm
- Define physics and reachability settings via `aas_settings_t`
- Provide entity-index conversion macros (`DF_AASENTNUMBER`, etc.)
- Pull in all AAS subsystem headers when compiled inside the botlib (guarded by `BSPCINCLUDE`)

## External Dependencies

- `aasfile.h` (implied): provides `aas_bbox_t`, `aas_vertex_t`, `aas_plane_t`, `aas_edge_t`, `aas_face_t`, `aas_area_t`, `aas_areasettings_t`, `aas_reachability_t`, `aas_node_t`, `aas_portal_t`, `aas_cluster_t`, and related index/count types — defined elsewhere
- `be_aas_funcs.h` / `be_aas.h` (implied): provides `aas_entityinfo_t`, `aas_trace_t`, `aas_areainfo_t`, `bot_entitystate_t`, `MAX_TRAVELTYPES` — defined elsewhere
- `q_shared.h` (implied): `vec3_t`, `qboolean`, `MAX_QPATH`, `byte` — defined elsewhere
- All `be_aas_*.h` subsystem headers: forward-declare the internal and public AAS API; implementations are defined in their corresponding `.c` files

# code/botlib/be_aas_entity.c
## File Purpose
Manages the AAS (Area Awareness System) entity table for the Quake III botlib. It synchronizes game-world entity state into the AAS spatial database, maintaining entity-to-AAS-area and entity-to-BSP-leaf linkages so the bot pathfinding system can reason about dynamic objects.

## Core Responsibilities
- Accept per-frame entity state updates and mirror them into `aasworld.entities[]`
- Detect changes in origin, angles, or bounding box to trigger spatial relinking
- Link/unlink entities to AAS areas and BSP leaf nodes
- Provide read accessors for entity origin, size, type, model index, and BSP data
- Iterate over valid entities (`AAS_NextEntity`) and find nearest entity by model
- Reset or invalidate the entity table between map loads or frames

## External Dependencies
- `../game/q_shared.h` — math macros (`VectorCopy`, `VectorAdd`, etc.), `qboolean`, `vec3_t`
- `be_aas_def.h` — `aasworld` global, `aas_entity_t`, `aas_world_t`
- `be_aas_funcs.h` — `AAS_UnlinkFromAreas`, `AAS_UnlinkFromBSPLeaves`, `AAS_LinkEntityClientBBox`, `AAS_BSPLinkEntity`, `AAS_BSPModelMinsMaxsOrigin`, `AAS_BestReachableLinkArea`, `AAS_Time`
- `be_interface.h` — `botimport` global
- `../game/botlib.h` — `BLERR_*` error codes, `bot_entitystate_t`, `PRESENCE_NORMAL`
- `../game/be_aas.h` — `bsp_entdata_t`, `aas_entityinfo_t` public types
- `aasfile.h` — AAS file format constants (included transitively)

# code/botlib/be_aas_entity.h
## File Purpose
Public and internal header for the AAS (Area Awareness System) entity subsystem within the Quake III botlib. It declares functions for querying and managing game entity state as it pertains to bot navigation and collision detection.

## Core Responsibilities
- Declares internal (AASINTERN-gated) entity lifecycle management functions (invalidate, unlink, reset, update)
- Exposes public API for querying entity spatial properties (origin, size, bounding box)
- Provides entity-to-AAS-area mapping for bot navigation queries
- Exposes entity type and model index accessors used by the bot AI layer

## External Dependencies
- `bot_entitystate_t` — defined in botlib/botlib.h or game interface headers
- `bsp_entdata_t` — defined in BSP/collision subsystem headers
- `aas_entityinfo_t` — defined in AAS internal headers (be_aas_def.h or aasfile.h)
- `vec3_t` — defined in q_shared.h

# code/botlib/be_aas_file.c
## File Purpose
Handles loading and writing of AAS (Area Awareness System) binary navigation files for the Quake III botlib. It reads the lump-based file format into the global `aasworld` structure and performs endian byte-swapping to ensure portability across architectures.

## Core Responsibilities
- Open, validate, and parse AAS files from disk into `aasworld` global state
- Perform little-endian byte-swapping on all loaded AAS data structures
- Free all AAS world data arrays and reset state flags (`AAS_DumpAASData`)
- Write in-memory AAS data back to disk in the lump-based format
- Obfuscate/deobfuscate the file header using a simple XOR cipher (`AAS_DData`)
- Validate AAS file identity, version, and BSP checksum against the loaded map

## External Dependencies
- `aasfile.h` — `AASID`, `AASVERSION`, `AASVERSION_OLD`, `AASLUMP_*` constants, `aas_header_t`, lump type definitions
- `be_aas_def.h` — defines `aasworld` global of type `aas_world_t`
- `be_interface.h` — `botimport` (FS I/O, print); `AAS_Error`
- `be_aas_funcs.h` — `AAS_Error` declaration
- `l_libvar.h` — `LibVarGetString` (reads `sv_mapChecksum`)
- `l_memory.h` — `GetClearedHunkMemory`, `FreeMemory`
- `q_shared.h` — `LittleLong`, `LittleFloat`, `LittleShort`, `fileHandle_t`, `qboolean`, `FS_READ/WRITE/SEEK_SET`, `Com_Memset`
- `botimport` (defined elsewhere) — virtual filesystem and print callbacks used throughout

# code/botlib/be_aas_file.h
## File Purpose
Header file declaring internal AAS (Area Awareness System) file I/O operations for the Quake III bot library. All declarations are gated behind the `AASINTERN` preprocessor guard, restricting visibility to internal botlib compilation units only.

## Core Responsibilities
- Declare the AAS file load interface
- Declare the AAS file write interface
- Declare AAS data cleanup and diagnostic utilities
- Enforce internal-only access via `AASINTERN` guard

## External Dependencies
- `AASINTERN` macro — must be defined by the including translation unit to expose these declarations.
- `qboolean` — defined in `q_shared.h` (engine shared types).
- Implementation: `code/botlib/be_aas_file.c`

# code/botlib/be_aas_funcs.h
## File Purpose
A convenience aggregation header for the AAS (Area Awareness System) subsystem of the Quake III botlib. It acts as a single-include facade that pulls in all AAS sub-module headers, conditional on not being compiled as part of the BSPC map compiler tool.

## Core Responsibilities
- Aggregates all AAS sub-module public (and internal) headers into one include
- Guards inclusion behind `#ifndef BSPCINCLUDE` to prevent use in the BSPC offline tool
- Provides a single include point for any translation unit needing full AAS API access

## External Dependencies
- All dependencies are local botlib headers listed above
- The `BSPCINCLUDE` macro is defined externally by the BSPC build system; its absence enables the includes
- The `AASINTERN` macro (used inside several bundled headers) gates internal-only declarations for botlib-internal translation units vs. external callers

# code/botlib/be_aas_main.c
## File Purpose
This is the main AAS (Area Awareness System) subsystem coordinator for Quake III's bot library. It manages the lifecycle of the AAS world — initialization, per-frame updates, map loading, and shutdown — and provides utility functions for string/model index lookups within the AAS world state.

## Core Responsibilities
- Lifecycle management: setup, load, init, per-frame update, shutdown of the AAS world
- Map loading: orchestrates loading of BSP and AAS files on map change
- Deferred initialization: drives incremental reachability and routing computation across frames
- String/model index registry: bidirectional lookup between config string indices and model names
- Developer diagnostics: exposes routing cache, memory usage, and memory dump via lib vars
- Routing cache persistence: triggers save of routing cache to disk on demand

## External Dependencies
- `q_shared.h`: `vec3_t`, `qboolean`, `Com_Memset`, `Com_sprintf`, `Q_stricmp`, `VectorSubtract`, `VectorNormalize`, `VectorMA`, `DotProduct`
- `l_memory.h`: `GetMemory`, `GetClearedHunkMemory`, `FreeMemory`, `PrintUsedMemorySize`, `PrintMemoryLabels`
- `l_libvar.h`: `LibVar`, `LibVarValue`, `LibVarGetValue`, `LibVarSet`
- `be_aas_def.h`: `aas_t`, `aas_entity_t` struct definitions (defined elsewhere)
- `be_interface.h`: `botimport` (defined elsewhere) — engine callback table for printing, file I/O, etc.; `bot_developer` flag
- Subsystem functions (defined elsewhere): `AAS_ContinueInitReachability`, `AAS_InitClustering`, `AAS_Optimize`, `AAS_WriteAASFile`, `AAS_InitRouting`, `AAS_UnlinkInvalidEntities`, `AAS_InvalidateEntities`, `AAS_ResetEntityLinks`, `AAS_LoadBSPFile`, `AAS_LoadAASFile`, `AAS_DumpBSPData`, `AAS_FreeRoutingCaches`, `AAS_InitAASLinkHeap`, `AAS_FreeAASLinkHeap`, `AAS_InitAASLinkedEntities`, `AAS_FreeAASLinkedEntities`, `AAS_DumpAASData`, `AAS_InitReachability`, `AAS_InitAlternativeRouting`, `AAS_ShutdownAlternativeRouting`, `AAS_InitSettings`, `AAS_WriteRouteCache`, `AAS_RoutingInfo`

# code/botlib/be_aas_main.h
## File Purpose
Public and internal header for the AAS (Area Awareness System) main module within Quake III's botlib. It declares the primary lifecycle functions for initializing, loading, updating, and shutting down the AAS world, as well as a small set of public utility queries.

## Core Responsibilities
- Guard internal AAS lifecycle functions behind the `AASINTERN` preprocessor gate
- Expose the global `aasworld` state to other internal AAS modules
- Declare public query functions usable outside the AAS internals (initialized state, loaded state, time, model index lookup)
- Declare a geometric utility function for projecting a point onto a line segment

## External Dependencies
- `aas_t` — defined in `be_aas_def.h` (included by internal files before this header)
- `vec3_t` — defined in `q_shared.h`
- `QDECL` — calling-convention macro from `q_shared.h`
- `AASINTERN` — preprocessor symbol defined by internal AAS compilation units

# code/botlib/be_aas_move.c
## File Purpose
Implements AAS (Area Awareness System) movement physics simulation for the Quake III bot library. It predicts client movement trajectories by simulating gravity, friction, acceleration, stepping, and liquid content detection. Results are used by the bot AI to evaluate reachability and plan navigation.

## Core Responsibilities
- Initialize AAS physics settings from library variables (`aassettings`)
- Detect ground contact, ladder proximity, and swimming state
- Simulate multi-frame client movement with full physics (gravity, friction, acceleration, stepping, crouching, jumping)
- Report movement stop-events (hit ground, enter liquid, enter area, fall damage, gap, bounding-box collision)
- Calculate horizontal velocity required for a jump arc between two points
- Calculate Z-velocity resulting from rocket/BFG self-damage jumps

## External Dependencies
- `../game/q_shared.h` — math types, vector macros, `qboolean`
- `l_libvar.h` — `LibVarValue` for reading physics cvars
- `be_aas_funcs.h` — `AAS_Trace`, `AAS_TraceClientBBox`, `AAS_TraceAreas`, `AAS_PointAreaNum`, `AAS_PointContents`, `AAS_PlaneFromNum`, `AAS_PointPresenceType`, `AAS_PresenceTypeBoundingBox`, `AAS_PointInsideFace`
- `be_aas_def.h` — `aasworld` global, `aas_settings_t`, area/face/plane data structures, flag constants (`AREA_LADDER`, `FACE_LADDER`, `PRESENCE_*`, `AREACONTENTS_*`, `SE_*`)
- `../game/botlib.h` — `botlib_import_t`, `botimport` (print/debug I/O)
- `be_aas_debug.h` (implicit via funcs) — `AAS_DebugLine`, `AAS_ClearShownDebugLines` (defined elsewhere)
- `AngleVectors`, `VectorNormalize`, `DotProduct`, `Com_Memset` — defined in `q_shared.c` / math libraries

# code/botlib/be_aas_move.h
## File Purpose
Public header for the AAS (Area Awareness System) movement prediction subsystem within the Quake III botlib. It declares functions used to simulate and predict client movement physics for bot navigation, including ground checks, swimming, ladder detection, and weapon-assisted jumping.

## Core Responsibilities
- Expose movement prediction API (`AAS_PredictClientMovement`, `AAS_ClientMovementHitBBox`)
- Provide terrain/environment query utilities (ground, water, ladder detection)
- Expose weapon-jump velocity calculators (rocket jump, BFG jump)
- Declare jump arc/trajectory helpers for reachability computation
- Conditionally expose internal `aassettings` global to other AAS modules

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `aas_clientmove_s`, `aas_reachability_s`, `aas_settings_t` — defined in `be_aas_def.h`
- `AASINTERN` — preprocessor guard defined by internal AAS translation units only

# code/botlib/be_aas_optimize.c
## File Purpose
Post-processes the AAS (Area Awareness System) world data after reachability calculation by stripping all geometric data (vertices, edges, faces) except those marked with `FACE_LADDER`. This reduces the in-memory and on-disk AAS footprint to only the geometry still needed at runtime.

## Core Responsibilities
- Allocate parallel optimized arrays for all AAS geometric primitives
- Selectively retain only ladder faces (and their referenced edges/vertices)
- Remap old indices to new compacted indices via lookup tables
- Preserve sign conventions (face/edge side/direction) throughout remapping
- Patch reachability records to reference the new compacted indices
- Replace `aasworld` geometry arrays with the compacted versions and free the old ones

## External Dependencies
- `q_shared.h` — `VectorCopy`, `Com_Memcpy`, base types
- `l_memory.h` — `GetClearedMemory`, `FreeMemory`
- `aasfile.h` — `aas_vertex_t`, `aas_edge_t`, `aas_face_t`, `aas_area_t`, `aas_edgeindex_t`, `aas_faceindex_t`, `FACE_LADDER`
- `be_aas_def.h` — `aasworld` global (defined elsewhere), `aas_reachability_t`, `TRAVEL_ELEVATOR`, `TRAVEL_JUMPPAD`, `TRAVEL_FUNCBOB`, `TRAVELTYPE_MASK`
- `be_interface.h` — `botimport` (defined elsewhere)

# code/botlib/be_aas_optimize.h
## File Purpose
Public header for the AAS (Area Awareness System) optimization module. It exposes a single entry point used to post-process and optimize AAS world data after loading or compilation.

## Core Responsibilities
- Declare the public interface for AAS optimization
- Expose `AAS_Optimize` as the sole external entry point for AAS data compaction/cleanup

## External Dependencies
- No includes in this header.
- `AAS_Optimize` is defined in `code/botlib/be_aas_optimize.c` (defined elsewhere).
- Consumers: AAS load/init routines in `be_aas_main.c` or `be_aas_file.c`.

# code/botlib/be_aas_reach.c
## File Purpose
Computes all inter-area reachability links for the AAS (Area Awareness System) navigation graph. It classifies every possible movement transition between adjacent AAS areas (walk, jump, swim, ladder, teleport, elevator, etc.) and stores the results so the bot pathfinder can later query travel costs and start/end points.

## Core Responsibilities
- Allocate and manage a fixed-size heap of temporary `aas_lreachability_t` link objects during calculation.
- Detect and create reachability links for every movement type: swim, equal-floor walk, step, barrier jump, water jump, walk-off-ledge, jump, ladder, teleport, elevator, func_bobbing, jump pad, grapple hook, and weapon jump.
- Iterate over all area pairs across multiple frames (`AAS_ContinueInitReachability`) to spread CPU cost.
- Mark areas adjacent to high-value items as valid weapon-jump targets (`AAS_SetWeaponJumpAreaFlags`).
- Finalize calculation by converting linked `aas_lreachability_t` lists into the compact `aasworld.reachability` array via `AAS_StoreReachability`.

## External Dependencies
- `../game/q_shared.h` — math types, `qboolean`, vector macros
- `l_log.h`, `l_memory.h`, `l_libvar.h`, `l_precomp.h`, `l_struct.h` — botlib utilities
- `aasfile.h`, `be_aas_def.h` — `aasworld` global, AAS data structure definitions
- `../game/botlib.h`, `../game/be_aas.h`, `be_aas_funcs.h` — travel-type constants, presence types, BSP query APIs
- **Defined elsewhere:** `aasworld` (global singleton), `aassettings`, `botimport`, `AAS_TraceClientBBox`, `AAS_PredictClientMovement`, `AAS_ClientMovementHitBBox`, `AAS_HorizontalVelocityForJump`, `AAS_RocketJumpZVelocity`, `AAS_BFGJumpZVelocity`, `AAS_PointAreaNum`, `AAS_LinkEntityClientBBox`, `AAS_UnlinkFromAreas`, `AAS_TraceAreas`, `AAS_PointInsideFace`, `AAS_PointContents`, `AAS_AreaPresenceType`, `AAS_DropToFloor`, `AAS_BSPModelMinsMaxsOrigin`, `AAS_NextBSPEntity`, `AAS_ValueForBSPEpairKey`, `AAS_VectorForBSPEpairKey`, `AAS_FloatForBSPEpairKey`, `AAS_IntForBSPEpairKey`, `AAS_PermanentLine`, `AAS_Trace`, `Sys_MilliSeconds`

# code/botlib/be_aas_reach.h
## File Purpose
Public and internal interface header for the AAS (Area Awareness System) reachability subsystem of Quake III Arena's bot library. It declares functions for querying area traversal properties and computing reachability relationships between AAS areas.

## Core Responsibilities
- Declare initialization and incremental computation of area reachabilities (internal only)
- Expose area property queries (swim, liquid, lava, slime, crouch, grounded, ladder, jump pad, do-not-enter)
- Provide spatial queries to find the best reachable area from a given origin/bounding box
- Support jump pad reachability queries
- Provide model-based reachability iteration

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `aas_link_t` — defined in `be_aas_def.h` (internal AAS structure)
- Implementation: `be_aas_reach.c`
- Consumers: `be_aas_route.c`, `be_ai_goal.c`, `be_ai_move.c`

# code/botlib/be_aas_route.c
## File Purpose
Implements the AAS (Area Awareness System) pathfinding and routing subsystem for the Quake III bot library. It computes and caches travel times between areas and portals using a Dijkstra-like relaxation algorithm over the cluster/portal graph. It also provides route prediction and nearest-hide-area queries for AI decision-making.

## Core Responsibilities
- Initialize and tear down all routing data structures (area/portal caches, travel time tables, reversed reachability links)
- Compute intra-cluster travel times via `AAS_UpdateAreaRoutingCache` (BFS/relaxation over reversed reachability)
- Compute inter-cluster travel times via `AAS_UpdatePortalRoutingCache` (portal-level routing)
- Manage an LRU-ordered routing cache with size limits; evict oldest entries when memory is low
- Serialize/deserialize pre-computed routing caches to/from `.rcd` files
- Expose `AAS_AreaTravelTimeToGoalArea` and `AAS_AreaReachabilityToGoalArea` as the primary bot query API
- Predict a route step-by-step with configurable stop events (`AAS_PredictRoute`)
- Find the nearest area hidden from an enemy (`AAS_NearestHideArea`)

## External Dependencies
- **Includes:** `q_shared.h`, `l_utils.h`, `l_memory.h`, `l_log.h`, `l_crc.h`, `l_libvar.h`, `l_script.h`, `l_precomp.h`, `l_struct.h`, `aasfile.h`, `botlib.h`, `be_aas.h`, `be_aas_funcs.h`, `be_interface.h`, `be_aas_def.h`
- **Defined elsewhere:** `aasworld` (global AAS world state), `botimport` (engine I/O interface), `bot_developer` (cvar), `AAS_Time`, `AAS_TraceAreas`, `AAS_TraceClientBBox`, `AAS_PointAreaNum`, `AAS_AreaReachability`, `AAS_AreaCrouch`, `AAS_AreaSwim`, `AAS_AreaGroundFaceArea`, `AAS_AreaDoNotEnter`, `AAS_ProjectPointOntoVector`, `AAS_AreaVisible`, `LibVarValue`, `Sys_MilliSeconds`, `CRC_ProcessString`, `GetMemory`, `GetClearedMemory`, `FreeMemory`, `AvailableMemory`, `Log_Write`

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

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `aas_reachability_s`, `aas_predictroute_s` — defined in `be_aas_def.h` or `aasfile.h`
- Travel flag constants (`TFL_*`) — defined in `be_aas_move.h` or `aasfile.h`
- Implementation: `be_aas_route.c`

# code/botlib/be_aas_routealt.c
## File Purpose
Implements alternative routing goal discovery for the AAS (Area Awareness System) bot navigation library. It identifies mid-range waypoint areas between a start and goal position that a bot could route through to take paths different from the shortest route.

## Core Responsibilities
- Identify "mid-range" AAS areas that lie geometrically between start and goal positions using travel-time thresholds
- Flood-fill connected mid-range areas into spatial clusters via `AAS_AltRoutingFloodCluster_r`
- Select one representative area per cluster (closest to cluster centroid) as an alternative route goal
- Populate an output array of `aas_altroutegoal_t` structs with alternative waypoints
- Manage lifecycle (init/shutdown) of working buffers `midrangeareas` and `clusterareas`

## External Dependencies
- `q_shared.h` — vector math macros, `qboolean`, `Com_Memset`
- `l_memory.h` — `GetMemory`, `FreeMemory`
- `l_log.h` — `Log_Write`
- `be_aas_def.h` — `aasworld` global (type `aas_t`), `aas_area_t`, `aas_face_t`
- `be_aas_funcs.h` — `AAS_AreaTravelTimeToGoalArea`, `AAS_AreaReachability`, `AAS_ShowAreaPolygons` (debug)
- `be_interface.h` — `botimport` (used in debug timing path only)
- `aasfile.h` — AAS file structures (`aas_area_t`, `aas_face_t`, area content flags)
- `botlib.h` / `be_aas.h` — `aas_altroutegoal_t`, `ALTROUTEGOAL_*` constants
- `aasworld` — defined in `be_aas_main.c` (external global)

# code/botlib/be_aas_routealt.h
## File Purpose
Public and internal interface header for the AAS (Area Awareness System) alternative routing subsystem. It exposes functions for computing alternative route goals between two AAS areas, used by the bot AI to find tactically varied paths.

## Core Responsibilities
- Declares internal (`AASINTERN`) lifecycle functions for the alternative routing system
- Exposes the public API for querying alternative route goals to bot clients
- Guards internal symbols behind the `AASINTERN` preprocessor gate, enforcing module encapsulation

## External Dependencies
- `aas_altroutegoal_t` — defined in `aasfile.h` or `be_aas_def.h`
- `vec3_t` — defined in `q_shared.h`
- `AASINTERN` — preprocessor macro controlling visibility of internal symbols
- Implementation resides in `be_aas_routealt.c` (defined elsewhere)

# code/botlib/be_aas_sample.c
## File Purpose
Implements AAS (Area Awareness System) environment sampling for the Quake III bot library. It provides spatial queries against the AAS BSP tree — point-to-area lookup, line tracing, bounding box area enumeration, entity linking/unlinking, and face containment tests used by bot navigation.

## Core Responsibilities
- Map points to AAS area numbers via BSP tree traversal
- Trace a bounding-box sweep through the AAS tree (`AAS_TraceClientBBox`)
- Collect all AAS areas a line segment passes through (`AAS_TraceAreas`)
- Link/unlink game entities into AAS areas for collision queries
- Manage a fixed-size free-list heap of `aas_link_t` nodes
- Test whether a point lies inside a face polygon
- Return area metadata (presence type, cluster, bounding box)

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `DotProduct`, `VectorCopy`, `VectorSubtract`, `VectorMA`, `VectorNormalize`, `Com_Memset`, `qboolean`
- `l_memory.h` — `GetHunkMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_libvar.h` (non-BSPC) — `LibVarValue` to read `max_aaslinks` cvar
- `be_aas_def.h` — `aasworld` global (type `aas_world_t`), `aas_link_t`, `aas_node_t`, `aas_plane_t`, `aas_face_t`, `aas_edge_t`, `aas_area_t`, `aas_areasettings_t`
- `be_aas_funcs.h` — `AAS_EntityCollision`, `AAS_AreaReachability` (defined elsewhere)
- `be_interface.h` — `bot_developer` flag (defined elsewhere)
- `botlib_import_t botimport` — engine printing/import functions (defined in `be_interface.c`)

# code/botlib/be_aas_sample.h
## File Purpose
Public and internal header for AAS (Area Awareness System) spatial sampling and querying operations. It exposes functions for point/bbox/trace queries against AAS geometry, and conditionally exposes internal link management functions when compiled with `AASINTERN` defined.

## Core Responsibilities
- Declare presence-type bounding box queries
- Declare point-in-area and area cluster/presence lookups
- Declare client bounding-box trace operations against AAS space
- Declare multi-area trace and bbox overlap enumeration
- Guard internal AAS link heap and entity linking functions behind `AASINTERN`

## External Dependencies
- `be_aas_def.h` or equivalent (defines `aas_face_t`, `aas_plane_t`, `aas_link_t`, `aas_trace_t`, `aas_areainfo_t`)
- `q_shared.h` (defines `vec3_t`, `qboolean`)
- All function bodies defined in `code/botlib/be_aas_sample.c`

# code/botlib/be_ai_char.c
## File Purpose
Implements the bot character system for Quake III Arena's botlib, loading and managing personality profiles (characteristics) from script files. Each bot character is a named collection of up to 80 typed key-value slots (integer, float, or string) associated with a skill level.

## Core Responsibilities
- Load bot character files from disk, parsing skill-bracketed blocks via the precompiler/script system
- Cache loaded characters in a global handle-indexed table to avoid redundant file I/O
- Apply default characteristics from a fallback character file when slots are uninitialized
- Interpolate numeric characteristics between two skill-level characters to produce fractional-skill variants
- Provide typed accessor functions (float, bounded float, integer, bounded integer, string) for game-side queries
- Free and shut down character resources, with optional reload-on-free behavior gated by a libvar

## External Dependencies
- `q_shared.h` — core types, `MAX_CLIENTS`, `MAX_QPATH`, `qboolean`, string utilities
- `l_log.h` / `Log_Write` — debug character dump output
- `l_memory.h` / `GetMemory`, `GetClearedMemory`, `FreeMemory` — botlib heap allocator
- `l_script.h` / `l_precomp.h` — lexer and precompiler (`LoadSourceFile`, `PC_ReadToken`, etc.)
- `l_libvar.h` / `LibVarGetValue` — runtime variable lookup (`bot_reloadcharacters`)
- `be_interface.h` — `botimport` global (print/error callbacks into the engine)
- `be_ai_char.h` — public interface declarations (defined elsewhere, exported from this file)
- `Sys_MilliSeconds` — timing macro used in `#ifdef DEBUG` path (defined in platform layer)

# code/botlib/be_ai_chat.c
## File Purpose
Implements the bot chat AI subsystem for Quake III Arena, managing bot console message queues, chat line selection, synonym/random-string expansion, match-template pattern matching, and reply-chat key evaluation. It provides the complete pipeline from raw console input through pattern matching to final chat message construction and delivery.

## Core Responsibilities
- Manage a fixed-size heap of `bot_consolemessage_t` nodes for per-bot console message queues
- Load and parse synonym, random-string, match-template, and reply-chat data files
- Match incoming strings against loaded `bot_matchtemplate_t` patterns and extract named variables
- Select and construct initial chat messages by type, with recency-avoidance logic
- Evaluate reply-chat key sets (AND/NOT/gender/name/string/variable) to choose best-priority reply
- Expand escape-coded chat message templates (`\x01v...\x01`, `\x01r...\x01`) with variable and random substitutions
- Deliver constructed messages via `EA_Command` (say / say_team / tell)

## External Dependencies
- `l_memory.h` — `GetMemory`, `GetClearedMemory`, `GetHunkMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_libvar.h` — `LibVarValue`, `LibVarString`, `LibVarGetValue`
- `l_script.h` / `l_precomp.h` — `source_t`, `token_t`, `LoadSourceFile`, `FreeSource`, `PC_ReadToken`, `PC_ExpectToken*`, `PC_CheckToken*`, `SourceError`, `SourceWarning`, `StripDoubleQuotes`
- `l_log.h` — `Log_FilePointer`, `Log_Write` (debug dump functions)
- `be_interface.h` — `botimport` (global import table for `Print`); `bot_developer` flag
- `be_aas.h` / `be_aas_funcs.h` — `AAS_Time()` (used for message recency timestamps)
- `be_ea.h` — `EA_Command` (delivers the final say/tell command to the game)
- `be_ai_chat.h` — public API declarations (`bot_match_t`, `MAX_MATCHVARIABLES`, `MAX_MESSAGE_SIZE`, gender constants, `BLERR_*`)
- `botlib.h` — `botimport_t` structure definition
- `q_shared.h` — `qboolean`, `MAX_CLIENTS`, `MAX_QPATH`, `Com_Memset`, `Com_Memcpy`, `Q_stricmp`, `Q_strncpyz`, `va`

# code/botlib/be_ai_gen.c
## File Purpose
Implements a fitness-proportionate (roulette wheel) genetic selection algorithm for the bot AI system. It provides utilities for selecting individuals from a ranked population, used to evolve bot behavior parameters over time.

## Core Responsibilities
- Perform weighted random selection from a ranked population (higher rank = higher probability)
- Fall back to uniform random selection when all rankings are zero or negative
- Select two parent bots and one child bot for genetic crossover, ensuring the child is selected inversely (lowest-ranked preferred)
- Enforce a hard cap of 256 bots for the parent/child selection function
- Validate minimum population size (at least 3 valid bots) before proceeding

## External Dependencies
- `../game/q_shared.h` — `random()` macro, `Com_Memcpy`, `qboolean`, `qtrue`/`qfalse`
- `be_interface.h` — `botimport` global (provides `botimport.Print`)
- `../game/botlib.h` — `PRT_WARNING` print type constant
- `l_memory.h`, `l_log.h`, `l_utils.h`, `l_script.h`, `l_precomp.h`, `l_struct.h`, `aasfile.h`, `be_aas_funcs.h`, `../game/be_aas.h`, `../game/be_ai_gen.h` — included but none of their symbols are directly used in this file's two functions; they establish the standard botlib compilation environment.
- `botimport` — defined in `be_interface.c` (global singleton).

# code/botlib/be_ai_goal.c
## File Purpose
Implements the bot goal AI subsystem for Quake III Arena. It manages level item tracking, per-bot goal stacks, and fuzzy-weight-based goal selection (both long-term and nearby goals) to drive bot navigation decisions.

## Core Responsibilities
- Load and manage item configuration (`items.c`) describing all pickup types in the level
- Build and maintain a runtime list of `levelitem_t` instances from BSP entities and live entity state
- Provide per-bot goal stacks (push/pop/query) via opaque integer handles
- Maintain per-bot avoid-goal lists with expiry times to prevent re-targeting recently visited items
- Select the best Long-Term Goal (LTG) and Near-By Goal (NBG) using fuzzy weight scoring divided by AAS travel time
- Parse `target_location` and `info_camp` BSP entities into map location and camp spot lists
- Track dynamically dropped entity items with timeout-based expiry

## External Dependencies
- `q_shared.h` — `vec3_t`, `qboolean`, `Com_Memset/Memcpy`, string utilities
- `l_libvar.h` — `LibVar`, `LibVarValue`, `LibVarString` (botlib config variables)
- `l_memory.h` — `GetClearedMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_log.h` — `Log_Write` (diagnostic logging)
- `l_script.h` / `l_precomp.h` — `LoadSourceFile`, `PC_ReadToken`, `PC_ExpectTokenType`, `FreeSource`, `SourceError`
- `l_struct.h` — `ReadStructure` (struct-driven config parsing)
- `be_aas_funcs.h` / `be_aas.h` — AAS queries (`AAS_AreaTravelTimeToGoalArea`, `AAS_BestReachableArea`, `AAS_PointAreaNum`, `AAS_Trace`, `AAS_NextBSPEntity`, `AAS_NextEntity`, `AAS_EntityInfo`, `AAS_Time`, etc.) — **defined in AAS subsystem**
- `be_ai_weight.h` — `FuzzyWeight`, `FuzzyWeightUndecided`, `ReadWeightConfig`, `FreeWeightConfig`, `FindFuzzyWeight`, `InterbreedWeightConfigs`, `EvolveWeightConfig` — **defined in be_ai_weight.c**
- `be_interface.h` — `botimport` (engine import struct for Print), `bot_developer` — **defined in be_interface.c**
- `be_ai_move.h` — `BotReachabilityArea` — **defined in be_ai_move.c**
- `be_ai_goal.h` — public API declarations (`bot_goal_t`, `MAX_GOALSTACK`, `MAX_AVOIDGOALS`, `GFL_*` flags, `BLERR_*` error codes)

# code/botlib/be_ai_move.c
## File Purpose
Implements the bot movement AI for Quake III Arena, translating high-level goal navigation into frame-by-frame elementary actions (EA_Move, EA_Jump, etc.) using the AAS (Area Awareness System) reachability graph. It manages per-bot movement state and handles all travel types from walking and jumping to grappling hooks and weapon jumps.

## Core Responsibilities
- Allocate, initialize, and free per-bot `bot_movestate_t` instances
- Determine which AAS reachability area a bot currently occupies
- Select the next reachability link toward a goal via routing and avoid-spot filtering
- Execute travel-type-specific movement logic (walk, crouch, jump, ladder, elevator, grapple, rocket/BFG jump, jump pad, func_bobbing, teleport, water jump)
- Manage reachability timeout, avoid-reach blacklisting, and avoid-spot hazard detection
- Initialize/shutdown the move AI subsystem, registering libvars for physics constants

## External Dependencies
- **AAS API** (`be_aas_funcs.h`, `be_aas.h`): `AAS_PointAreaNum`, `AAS_TraceAreas`, `AAS_TraceClientBBox`, `AAS_Trace`, `AAS_PredictClientMovement`, `AAS_ReachabilityFromNum`, `AAS_NextAreaReachability`, `AAS_AreaReachability`, `AAS_AreaTravelTimeToGoalArea`, `AAS_TravelFlagForType`, `AAS_Swimming`, `AAS_OnGround`, `AAS_AgainstLadder`, `AAS_JumpReachRunStart`, `AAS_OriginOfMoverWithModelNum`, `AAS_BSPModelMinsMaxsOrigin`, `AAS_NextModelReachability`, `AAS_EntityModelindex`, `AAS_EntityModelNum`, `AAS_EntityInfo`, `AAS_EntityType`, `AAS_NextEntity`, `AAS_Time`, `AAS_PointContents`, `AAS_HorizontalVelocityForJump`, `AAS_AreaPresenceType`, `AAS_AreaContentsTravelFlags`, `AAS_AreaDoNotEnter`, `AAS_AreaJumpPad`, `AAS_NextBSPEntity`, `AAS_ValueForBSPEpairKey`
- **EA API** (`be_ea.h`): `EA_Move`, `EA_Jump`, `EA_DelayedJump`, `EA_Crouch`, `EA_Walk`, `EA_Attack`, `EA_MoveForward`, `EA_MoveUp`, `EA_View`, `EA_SelectWeapon`, `EA_Command`
- **botimport**: `Print` for debug/error messages (defined in `be_interface.h`)
- **bot_developer**: External debug flag (defined elsewhere)
- `GetClearedMemory` / `FreeMemory`: from `l_memory.h`
- `LibVar`: from `l_libvar.h`

# code/botlib/be_ai_weap.c
## File Purpose
Implements the weapon AI subsystem for Q3 bots, responsible for loading weapon/projectile configuration data from script files, managing per-bot weapon state, and selecting the best weapon to use in combat via fuzzy-weight evaluation.

## Core Responsibilities
- Load and parse weapon configuration files (`weapons.c`) into `weaponconfig_t` structures
- Load per-bot fuzzy weight configurations for weapon selection scoring
- Map parsed weapon names to fuzzy weight indices via `WeaponWeightIndex`
- Allocate and free per-bot `bot_weaponstate_t` handles (one per client slot)
- Evaluate all valid weapons against a bot's inventory using fuzzy logic to select the best fight weapon
- Provide weapon info lookup by weapon number for external callers
- Initialize and shut down the global weapon AI subsystem

## External Dependencies
- `l_script.h` / `l_precomp.h` — `LoadSourceFile`, `PC_ReadToken`, `FreeSource`, `PC_SetBaseFolder`
- `l_struct.h` — `ReadStructure`, `WriteStructure`, `fielddef_t`, `structdef_t`
- `be_ai_weight.h` — `weightconfig_t`, `ReadWeightConfig`, `FreeWeightConfig`, `FindFuzzyWeight`, `FuzzyWeight`
- `be_interface.h` — `botimport` (global import table providing `Print`)
- `l_libvar.h` — `LibVarValue`, `LibVarString`, `LibVarSet`
- `l_memory.h` — `GetClearedMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `../game/be_ai_weap.h` — `weaponinfo_t`, `projectileinfo_t` type definitions (defined elsewhere)
- `botlib.h` — `BLERR_*` error codes, `MAX_CLIENTS`

# code/botlib/be_ai_weight.c
## File Purpose
Implements a fuzzy logic weight evaluation system for the Q3 bot AI, parsing hierarchical weight configuration files and evaluating weighted decisions based on bot inventory state. It supports both deterministic and randomized ("undecided") weight lookups, as well as genetic-algorithm-style evolution and interbreeding of weight configs.

## Core Responsibilities
- Parse `weight` config files into `weightconfig_t` trees of `fuzzyseperator_t` nodes
- Cache loaded weight configs in a global file list (`weightFileList`) to avoid redundant disk reads
- Evaluate fuzzy weights given a bot's inventory array (deterministic and stochastic variants)
- Support evolutionary mutation (`EvolveWeightConfig`) and blending (`InterbreedWeightConfigs`) of weight configs
- Free and shut down weight config memory on demand

## External Dependencies
- `l_precomp.h` / `l_script.h`: `source_t`, `token_t`, `PC_*` parsing functions, `LoadSourceFile`, `FreeSource`
- `l_memory.h`: `GetClearedMemory`, `FreeMemory`
- `l_libvar.h`: `LibVarGetValue` (reads `bot_reloadcharacters`)
- `be_interface.h`: `botimport` (print/error callbacks)
- `be_ai_weight.h`: Type definitions for `weightconfig_t`, `weight_t`, `fuzzyseperator_t`, `WT_BALANCE`, `MAX_WEIGHTS`
- `q_shared.h`: `qboolean`, `Q_strncpyz`, `random()`, `crandom()`
- `Sys_MilliSeconds` (DEBUG only): defined in platform layer

# code/botlib/be_ai_weight.h
## File Purpose
Defines the data structures and public API for the botlib's fuzzy logic weighting system. It provides a configuration-driven framework for evaluating weighted decisions based on bot inventory state, used by the AI goal and decision-making subsystems.

## Core Responsibilities
- Define the `fuzzyseperator_t` linked-list node for fuzzy logic interval separation
- Define `weight_t` (named weight entry) and `weightconfig_t` (full weight configuration) containers
- Declare I/O functions for loading, saving, and freeing weight configurations from disk
- Declare evaluation functions that compute fuzzy weight values from bot inventory
- Declare mutation/evolution utilities for weight configs (used in bot training/genetic-style tuning)

## External Dependencies
- `MAX_QPATH` — defined in `q_shared.h` (engine shared header)
- `qboolean` — engine boolean typedef from `q_shared.h`
- `WT_BALANCE` (`1`) — constant used to tag separator nodes of balance type; consumed by `be_ai_weight.c`
- `MAX_WEIGHTS` (`128`) — caps the static weight array in `weightconfig_t`
- Implementation: `be_ai_weight.c`

# code/botlib/be_ea.c
## File Purpose
Implements the Elementary Actions (EA) layer of the Quake III bot library, providing the lowest-level interface through which bots express input — movement, aiming, attacking, jumping, crouching, and chat commands. It translates high-level bot decisions into `bot_input_t` state buffers that are later consumed by the engine.

## Core Responsibilities
- Allocate and manage per-client `bot_input_t` input buffers
- Set action flags (attack, jump, crouch, walk, use, gesture, etc.) on bot input state
- Set movement direction, speed, and view angles
- Issue text-based client commands (say, say_team, tell, use item, drop item)
- Handle jump de-bounce logic via `ACTION_JUMPEDLASTFRAME`
- Expose `EA_GetInput` to retrieve accumulated input for a frame
- Expose `EA_ResetInput` to clear per-frame state while preserving jump carry-over

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `VectorCopy`, `VectorClear`, `Com_Memcpy`, `qboolean`
- `l_memory.h` — `GetClearedHunkMemory`, `FreeMemory`
- `../game/botlib.h` — `bot_input_t`, `ACTION_*` flags, `botimport` (struct of engine callbacks), `BLERR_NOERROR`
- `be_interface.h` — `botlibglobals` (provides `maxclients`)
- `botimport.BotClientCommand` — engine callback; defined in the engine, not in this file
- `va()` — defined in `q_shared.c`

# code/botlib/be_interface.c
## File Purpose
This is the primary entry point and export layer for Quake III Arena's bot library (botlib). It initializes, shuts down, and exposes all bot subsystem APIs to the engine via a versioned function-pointer table (`botlib_export_t`), and bridges the engine's import callbacks into the library's global `botimport`.

## Core Responsibilities
- Implement `GetBotLibAPI`, the single DLL/module entry point that validates API version and returns `botlib_export_t`
- Initialize all bot subsystems in order (AAS, EA, WeaponAI, GoalAI, ChatAI, MoveAI) via `Export_BotLibSetup`
- Shut down all subsystems and free all resources via `Export_BotLibShutdown`
- Validate client/entity numbers before forwarding calls to internal subsystems
- Populate the three nested export structs (AAS, EA, AI) with function pointers to internal implementations
- Expose libvar get/set, precompiler handle functions, frame ticking, and map loading
- Provide a debug-only `BotExportTest` hook for in-engine AAS visualization

## External Dependencies
- `../game/botlib.h` — `botlib_export_t`, `botlib_import_t`, `BOTLIB_API_VERSION`, error codes
- `../game/be_aas.h`, `be_aas_funcs.h`, `be_aas_def.h` — AAS query functions and `aasworld` global
- `../game/be_ea.h` — Elementary action functions
- `../game/be_ai_*.h` — Higher-level AI subsystem APIs
- `be_ai_weight.h` — `BotShutdownWeights`
- `l_libvar.h` — Library variable system
- `l_precomp.h` — Precompiler (PC_*) functions; defined in `l_precomp.c`
- `l_log.h` — Log file; defined in `l_log.c`
- `aasworld` (global `aas_t`) — Defined in `be_aas_main.c`/`be_aas_def.h`; accessed directly in `BotExportTest`

# code/botlib/be_interface.h
## File Purpose
Declares the global state and external symbols for the botlib interface layer. It defines the central `botlib_globals_t` structure that tracks top-level botlib runtime state, and exposes key extern declarations used across the botlib subsystem.

## Core Responsibilities
- Define the `botlib_globals_t` struct holding library-wide runtime state
- Expose the `botlibglobals` singleton and `botimport` interface externals
- Expose the `bot_developer` flag for conditional debug/verbose behavior
- Declare the `Sys_MilliSeconds` platform timing function
- Gate optional debug fields (`debug`, `goalareanum`, `goalorigin`, `runai`) behind `#ifdef DEBUG`
- Enable `RANDOMIZE` macro to vary bot decision-making behavior

## External Dependencies
- `botlib_import_t` — defined in `botlib.h` (the engine-to-botlib import function table); used here by extern declaration only
- `vec3_t`, `qboolean` — defined in `q_shared.h`; used only under `#ifdef DEBUG`
- `Sys_MilliSeconds` — implemented in platform-specific system files (not in botlib)

# code/botlib/l_crc.c
## File Purpose
Implements a 16-bit CCITT CRC (XMODEM variant) using polynomial 0x1021 for data integrity verification within the botlib subsystem. It provides both stateful (incremental) and stateless (one-shot) CRC computation over byte sequences.

## Core Responsibilities
- Initialize a CRC accumulator to the standard CCITT seed value (`0xffff`)
- Process individual bytes into a running CRC value via table lookup
- Process a complete byte string in one call, returning a finalized CRC
- Support incremental/continuation CRC computation across multiple string segments
- Finalize a CRC by XOR-ing with `CRC_XOR_VALUE` (0x0000, effectively a no-op here)

## External Dependencies
- `<stdlib.h>`, `<stdio.h>`, `<string.h>` — standard C library (included but not directly used in function bodies).
- `../game/q_shared.h` — provides the `byte` typedef.
- `../game/botlib.h` — botlib API types.
- `be_interface.h` — provides `botimport` (referenced in comment only; not actually called in this file's functions).

# code/botlib/l_crc.h
## File Purpose
Declares the public interface for a CRC (Cyclic Redundancy Check) checksum utility used within the botlib. Provides functions for computing and incrementally updating 16-bit CRC values over byte sequences.

## Core Responsibilities
- Define the `crc_t` type alias for 16-bit CRC values
- Expose CRC initialization, incremental byte/string processing, and value extraction functions

## External Dependencies
- `byte` type — defined elsewhere (expected from `q_shared.h` or equivalent botlib common header).
- No standard library includes are visible in this header.

# code/botlib/l_libvar.c
## File Purpose
Implements a lightweight key-value variable system ("libvars") internal to the bot library. It provides create, read, update, and delete operations for named variables that store both a string value and a precomputed float value, independent of the engine's main cvar system.

## Core Responsibilities
- Allocate and free individual `libvar_t` nodes from bot library heap
- Maintain a singly-linked global list of all active libvars
- Perform lazy creation: create a variable on first access with a default value
- Convert string values to floats via a custom parser (`LibVarStringValue`)
- Track a `modified` flag per variable so callers can poll for changes
- Provide bulk teardown of all libvars at bot library shutdown

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `qtrue`/`qfalse`, `Q_stricmp`, `Com_Memset`
- `l_memory.h` — `GetMemory`, `FreeMemory` (bot library heap wrappers)
- `strcpy`, `strlen` — C standard library (available because this code compiles outside the Q3VM)
- `libvar_t` defined in `l_libvar.h`

# code/botlib/l_libvar.h
## File Purpose
Declares the botlib's internal configuration variable system (`libvar`), a lightweight cvar-like mechanism used exclusively within the bot library to store and query named string/float settings without going through the engine's cvar system.

## Core Responsibilities
- Define the `libvar_t` linked-list node structure for named variables
- Declare lifecycle management (allocation/deallocation of all vars)
- Declare lookup functions (by name, returning struct, string, or float)
- Declare create-or-get helpers (`LibVar`, `LibVarValue`, `LibVarString`)
- Declare mutation (`LibVarSet`) and change-detection (`LibVarChanged`, `LibVarSetNotModified`) interfaces

## External Dependencies
- `qboolean` — defined in `q_shared.h` (engine shared types); not defined in this file.
- Implementation body: `l_libvar.c` (defined elsewhere).
- No standard library headers included directly in this header.

# code/botlib/l_log.c
## File Purpose
Provides a simple file-based logging facility for the botlib subsystem. It manages a single global log file, supporting plain and timestamped write operations gated by the `"log"` library variable.

## Core Responsibilities
- Open and close a single log file on demand, guarded by the `"log"` libvar
- Write formatted (variadic) messages to the log file
- Write timestamped, sequenced entries using `botlibglobals.time`
- Flush the log file buffer on demand
- Expose the raw `FILE*` pointer for external direct writes

## External Dependencies
- `<stdlib.h>`, `<stdio.h>`, `<string.h>` — standard C I/O and string functions
- `../game/q_shared.h` — shared engine types (`QDECL`, etc.)
- `../game/botlib.h` — `PRT_MESSAGE`, `PRT_ERROR` print type constants
- `be_interface.h` — `botimport` (for `botimport.Print`) and `botlibglobals` (for `botlibglobals.time`)
- `l_libvar.h` — `LibVarValue` (defined in `l_libvar.c`)
- `botimport.Print` — defined elsewhere (host engine), called via function pointer
- `botlibglobals` — defined in `be_interface.c`

# code/botlib/l_log.h
## File Purpose
Public header for the botlib logging subsystem. It declares the interface for opening, writing to, flushing, and closing a log file used by the bot library during development and debugging.

## Core Responsibilities
- Declare the log file lifecycle API (open, close, shutdown)
- Declare formatted write functions (with and without timestamps)
- Expose the underlying `FILE*` for external consumers
- Declare a flush function to force buffered output to disk

## External Dependencies
- `<stdio.h>` — for the `FILE` type used in `Log_FilePointer`.
- `QDECL` — macro defined in `q_shared.h` (typically expands to `__cdecl` on Windows, empty on others).
- Implementation defined in `code/botlib/l_log.c`.

# code/botlib/l_memory.c
## File Purpose
Provides the botlib's memory allocation abstraction layer, delegating all actual allocations to the engine via the `botimport` function table. Supports two compile-time configurations: a full memory manager (`MEMORYMANEGER`) with block tracking, and a lightweight mode that only prepends a magic ID word.

## Core Responsibilities
- Wrap `botimport.GetMemory` and `botimport.HunkAlloc` with bookkeeping headers
- Validate pointers on free by checking magic IDs (`MEM_ID` / `HUNK_ID`)
- Track total allocated bytes, total botlib memory, and block count (manager mode only)
- Provide zeroed variants of both heap and hunk allocators
- Dump all live allocations and report memory usage (manager mode)
- Conditionally compile debug variants that record label, file, and line per allocation

## External Dependencies
- `../game/q_shared.h` — `Com_Memset`, basic types
- `../game/botlib.h` — `botlib_import_t` definition (provides `GetMemory`, `FreeMemory`, `HunkAlloc`, `AvailableMemory`, `Print`)
- `l_log.h` — `Log_Write` (used by `PrintMemoryLabels`)
- `be_interface.h` — `botimport` extern (the live `botlib_import_t` instance)
- `botimport` — defined in `be_interface.c`; all actual memory operations delegate through it

# code/botlib/l_memory.h
## File Purpose
Public interface for the botlib's internal memory management subsystem. It declares allocation/deallocation functions for both standard heap memory and hunk (engine-side) memory, with optional debug instrumentation via the `MEMDEBUG` preprocessor toggle.

## Core Responsibilities
- Declare heap allocation functions (`GetMemory`, `GetClearedMemory`)
- Declare hunk allocation functions (`GetHunkMemory`, `GetClearedHunkMemory`) or alias them to heap variants under `BSPC`
- Provide debug variants that capture allocation label, source file, and line number
- Macro-redirect allocation calls transparently when `MEMDEBUG` is defined
- Declare deallocation, introspection, and bulk-free utilities

## External Dependencies
- No includes in this file itself.
- Implementations in `l_memory.c` depend on engine trap functions or `malloc`/`free` (not inferable here).
- `BSPC` and `MEMDEBUG` — compile-time defines set by the build system.
- `FreeMemory`, `GetMemory`, etc. are consumed by virtually all other botlib `.c` files.

# code/botlib/l_precomp.c
## File Purpose
Implements a C-like preprocessor (precompiler) used by the botlib to parse configuration and script files. It handles `#define`, `#include`, `#ifdef`/`#ifndef`/`#if`/`#elif`/`#else`/`#endif`, macro expansion, and expression evaluation for conditional compilation directives.

## Core Responsibilities
- Load and manage a stack of script files (`source_t`), supporting `#include`
- Parse and store macro definitions (`define_t`) with optional parameters, using a hash table for fast lookup
- Expand macros (including stringizing `#` and token-merging `##` operators) into the token stream
- Evaluate constant integer/float expressions in `#if`/`#elif` directives
- Manage conditional compilation skip state via an indent stack
- Expose a handle-based API (`PC_LoadSourceHandle`, `PC_ReadTokenHandle`, etc.) for external consumers
- Maintain a global define list injected into every opened source

## External Dependencies
- `l_script.h` — `script_t`, `token_t`, `PS_ReadToken`, `LoadScriptFile`, `LoadScriptMemory`, `FreeScript`, `EndOfScript`, `StripDoubleQuotes`, `PS_SetBaseFolder`
- `l_memory.h` — `GetMemory`, `GetClearedMemory`, `FreeMemory`
- `l_log.h` — `Log_Write` (used in hash-table debug print, conditionally)
- `be_interface.h` (BOTLIB) — `botimport.Print` for error/warning output
- `q_shared.h` — `Com_Memcpy`, `Com_Memset`, `Com_Error`, `Q_stricmp`
- `time.h` — `time()`, `ctime()` for `__DATE__`/`__TIME__` builtins
- `PC_NameHash`, `PC_AddDefineToHash`, `PC_FindHashedDefine` — defined in this file, used throughout

# code/botlib/l_precomp.h
## File Purpose
Declares the public interface for the botlib's C-style preprocessor, which tokenizes and macro-expands script/config files used by the bot AI system. It provides `#define`, `#if`/`#ifdef`/`#ifndef`/`#else`/`#elif` conditional compilation, and `#include` support for bot script parsing.

## Core Responsibilities
- Define data structures for macro definitions (`define_t`), conditional indent tracking (`indent_t`), and source file state (`source_t`)
- Declare token reading and expectation functions used by higher-level bot script parsers
- Declare macro/define management (per-source and global)
- Declare source file loading from disk or memory
- Expose a handle-based API (`PC_LoadSourceHandle` etc.) for use via the engine's trap/syscall interface
- Provide cross-platform path separator macros and BSPC build compatibility shims

## External Dependencies
- `token_t`, `script_t`, `punctuation_t` — defined in `l_script.h`
- `MAX_QPATH` — defined in `q_shared.h`
- `QDECL` — defined in `q_shared.h` (or stubbed for BSPC)
- `pc_token_t` — defined in `q_shared.h` when not building BSPC, or locally here under the BSPC guard

# code/botlib/l_script.c
## File Purpose
Implements a reusable lexicographical (lexer/tokenizer) parser used by the Quake III bot library, BSP converter (BSPC), and MrElusive's QuakeC Compiler. It parses C-like script files into typed tokens (strings, numbers, names, punctuation) from either file or memory buffers.

## Core Responsibilities
- Load script text from disk (`LoadScriptFile`) or memory (`LoadScriptMemory`) into a `script_t` context
- Advance through whitespace and C/C++-style comments (`PS_ReadWhiteSpace`)
- Tokenize input into strings, literals, numbers (decimal/hex/octal/binary), identifiers, and punctuation
- Provide expect/check helpers for parser consumers to assert or conditionally consume tokens
- Support token unread (one-token pushback via `script->tokenavailable`)
- Route error/warning output to the correct backend (botlib, MEQCC, BSPC) via compile-time `#ifdef`

## External Dependencies
- `q_shared.h` — `Com_Memset`, `Com_Memcpy`, `Com_sprintf`, `COM_Compress`, `qboolean`, `MAX_QPATH`, file handle types (BOTLIB build)
- `botlib.h` / `be_interface.h` — `botimport` (for `Print`, `FS_FOpenFile`, `FS_Read`, `FS_FCloseFile`) (BOTLIB build)
- `l_memory.h` — `GetMemory`, `GetClearedMemory`, `FreeMemory` (defined in `l_memory.c`)
- `l_log.h` — `Log_Print` (BSPC build only)
- `COM_Compress` — defined in `qcommon/common.c` (or equivalent); strips comments/redundant whitespace from loaded buffer

# code/botlib/l_script.h
## File Purpose
Defines the public interface for a lexicographical script parser used by the botlib. It provides token-based parsing of text scripts and configuration files, supporting C/C++-style syntax including strings, literals, numbers (decimal, hex, octal, binary, float), and a comprehensive punctuation set.

## Core Responsibilities
- Define token types and subtypes for lexical classification
- Define punctuation symbol constants (P_*) for all C/C++ operators and delimiters
- Declare the `script_t` state structure representing a loaded script with cursor tracking
- Declare the `token_t` structure for individual parsed tokens
- Declare the `punctuation_t` structure for customizable punctuation tables
- Expose the full parser API: read, expect, check, skip, unread operations
- Provide script lifecycle functions: load from file/memory, reset, free

## External Dependencies
- No explicit includes shown; implementation (`l_script.c`) will pull in standard C I/O and string headers.
- `QDECL`: calling-convention macro defined in `q_shared.h` or conditionally as empty for BSPC builds.
- `BSPC`: build-time define selecting BSP compiler context over botlib context.
- `LoadScriptFile` depends on a file system abstraction defined elsewhere (likely `l_memory.h` / OS file I/O).

# code/botlib/l_struct.c
## File Purpose
Provides generic serialization and deserialization of C structures to/from botlib script sources and plain text files. It maps a runtime `structdef_t` schema (field names, types, offsets) onto raw memory, enabling data-driven config loading and saving without hand-written parsers per struct.

## Core Responsibilities
- Look up a named field within a `fielddef_t` array
- Parse numeric values (int, char, float) from a `source_t` token stream with range validation
- Parse character literals and quoted strings from token streams
- Recursively deserialize a brace-delimited block from a `source_t` into a flat memory buffer
- Write indentation, float values (trailing-zero stripped), and full structures to a `FILE*`
- Recursively serialize a structure to an indented text file

## External Dependencies
- `l_precomp.h` — `source_t`, `PC_ExpectAnyToken`, `PC_ExpectTokenString`, `PC_ExpectTokenType`, `PC_CheckTokenString`, `PC_UnreadLastToken`, `SourceError`
- `l_script.h` — `token_t`, `StripDoubleQuotes`, `StripSingleQuotes`, `TT_*` constants
- `l_struct.h` — `fielddef_t`, `structdef_t`, `MAX_STRINGFIELD`, `FT_*` constants (self-header)
- `l_utils.h` — `Maximum`, `Minimum` macros (defined elsewhere)
- `q_shared.h` — `qboolean`, `qtrue`, `qfalse` (game shared types)
- `be_interface.h` — included for botlib import table context; no direct calls visible here
- Standard C: `strcmp`, `strncpy`, `sprintf`, `strlen`, `fprintf` — standard library

# code/botlib/l_struct.h
## File Purpose
Defines a generic, data-driven framework for reading and writing arbitrary C structs from/to script files and disk. Field descriptors encode name, offset, type, and constraints, enabling reflection-like serialization of botlib configuration structures.

## Core Responsibilities
- Define field type constants (`FT_CHAR`, `FT_INT`, `FT_FLOAT`, `FT_STRING`, `FT_STRUCT`) and subtype modifier flags (`FT_ARRAY`, `FT_BOUNDED`, `FT_UNSIGNED`)
- Provide `fielddef_t` to describe a single field within a struct (name, byte offset, type info, bounds, nested struct pointer)
- Provide `structdef_t` to describe a complete struct (size + field array)
- Declare `ReadStructure` for deserializing a struct from a parsed script token stream
- Declare `WriteStructure` for serializing a struct to a `FILE*`
- Declare utility formatters `WriteIndent` and `WriteFloat`

## External Dependencies
- `source_t` — defined in `code/botlib/l_script.h` (script parser state)
- `struct structdef_s` — self-referential forward declaration within `fielddef_t.substruct` to support nested struct recursion
- `FILE*` — standard C `<stdio.h>`
- Implementation: `code/botlib/l_struct.c`

# code/botlib/l_utils.h
## File Purpose
A minimal utility header for the botlib subsystem providing convenience macro aliases. It maps botlib-local names to engine-standard symbols and defines simple arithmetic macros.

## Core Responsibilities
- Aliases `vectoangles` under a more descriptive macro name for botlib use
- Aliases `MAX_QPATH` under the platform-conventional `MAX_PATH` name
- Provides inline `Maximum` and `Minimum` comparison macros

## External Dependencies
- `vectoangles` — defined elsewhere (engine shared code / `q_math.c`); not declared here.
- `MAX_QPATH` — defined in `q_shared.h` or equivalent engine header; must be visible at inclusion time for `MAX_PATH` to resolve correctly.

# code/bspc/_files.c
## File Purpose
This is a pure documentation/manifest file for the BSPC (BSP Compiler) tool. It contains no executable code — its sole purpose is to enumerate and briefly describe every source file belonging to the BSPC module.

## Core Responsibilities
- Serves as a human-readable index of all `.c` source files in the BSPC tool
- Groups files by subsystem: AAS generation, map loading, BSP processing, BSP I/O libraries, GL utilities, and platform-specific Win32 modules
- Documents which files are ported/adapted from an external BSP compiler (marked with `(*)`)

## External Dependencies
None. The file contains only a block comment; there are no `#include` directives or symbol references.

---

**Notes on file groupings documented here:**

| Group | Files | Role |
|---|---|---|
| AAS generation | `aas_areamerging.c` … `aas_store.c` | Build the Area Awareness System navigation graph |
| Map loading | `map.c`, `map_hl/q1/q2/q3/sin.c` | Parse source map formats from multiple games |
| BSP processing `(*)` | `tree.c`, `brushbsp.c`, `portals.c`, `csg.c`, `leakfile.c`, `textures.c` | Core BSP compiler stages adapted from id's tools |
| BSP I/O libraries | `l_bsp_*.c`, `l_cmd/log/math/mem/poly/script/threads/utils/qfiles.c` | Low-level parsing, math, memory, and file I/O |
| GL debug output `(*)` | `gldraw.c`, `glfile.c`, `nodraw.c` | Optional OpenGL visualisation and draw-suppression |
| Win32 platform | `bspc.c`, `winbspc.c`, `win32_*.c` | Console/GUI entry points and OS-specific services |

# code/bspc/aas_areamerging.c
## File Purpose
Implements the area-merging pass of the BSPC (BSP Compiler) AAS (Area Awareness System) generation pipeline. It iterates over temporary AAS areas, tests adjacent area pairs for convexity compatibility, and merges qualifying pairs into a single new convex area to reduce the total area count.

## Core Responsibilities
- Test whether two faces from different areas would form a non-convex region if merged (`NonConvex`)
- Validate merge eligibility: matching presence type, contents, and model number
- Detect ground/gap face flag conflicts that would block a merge
- Construct a new merged `tmp_area_t` by adopting all non-separating faces from both source areas
- Mark source areas as invalid and point them to the merged area via `mergedarea`
- Drive a two-phase merge loop: grounded areas first, then all areas, until no further merges occur
- Refresh the BSP tree's leaf pointers to follow `mergedarea` chains after merging

## External Dependencies
- `qbsp.h` — `mapplanes`, `plane_t`, `winding_t`, `DotProduct`, `Error`, `qprintf`
- `aasfile.h` — `FACE_GROUND`, `FACE_GAP` flag constants
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmp_aas_t`, `tmpaasworld`; functions `AAS_AllocTmpArea`, `AAS_RemoveFaceFromArea`, `AAS_AddFaceSideToArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`, `AAS_FlipAreaFaces`, `AAS_GapFace`
- `aas_store.h` — included transitively via `aas_create.h`; `aasworld` global
- `Log_Write` — defined elsewhere (logging utility from `l_log.c`)

# code/bspc/aas_areamerging.h
## File Purpose
Header file declaring the public interface for the AAS (Area Awareness System) area merging pass within the BSPC (BSP Compiler) tool. It exposes a single entry-point function used during AAS world generation to reduce area count by combining adjacent compatible areas.

## Core Responsibilities
- Declares the `AAS_MergeAreas` function as a public symbol for use by other BSPC compilation units

## External Dependencies
- No includes in this header; implementation dependencies are in `aas_areamerging.c`
- `AAS_MergeAreas` is defined elsewhere (`aas_areamerging.c`)

# code/bspc/aas_cfg.c
## File Purpose
Manages the AAS (Area Awareness System) configuration for the BSPC map compiler tool. It defines, loads, and applies physics and reachability settings used during AAS file generation from BSP maps.

## Core Responsibilities
- Define field descriptor tables (`fielddef_t`) and struct descriptors (`structdef_t`) for `cfg_t` and `aas_bbox_t` using offset macros
- Provide default Q3A configuration values via `DefaultCfg()`
- Parse a `.cfg` file using the botlib precompiler to populate the global `cfg` struct
- Validate loaded configuration (gravity direction magnitude, bounding box count)
- Propagate loaded float config values into the botlib libvar system via `SetCfgLibVars()`
- Provide a `va()` varargs string formatting utility

## External Dependencies
- `qbsp.h` — BSPC-wide types and declarations
- `float.h` — `FLT_MAX`
- `../botlib/aasfile.h` — `aas_bbox_t`, presence type constants (`PRESENCE_NORMAL`, `PRESENCE_CROUCH`)
- `aas_store.h` — `AAS_MAX_BBOXES`
- `aas_cfg.h` — `cfg_t`, `cfg` extern declaration
- `../botlib/l_precomp.h` — `source_t`, `token_t`, `LoadSourceFile`, `FreeSource`, `PC_ReadToken`, `SourceError`, `SourceWarning`
- `../botlib/l_struct.h` — `fielddef_t`, `structdef_t`, `ReadStructure`, `FT_FLOAT`, `FT_INT`, `FT_ARRAY`, `FT_TYPE`
- `../botlib/l_libvar.h` — `LibVarSet` (defined elsewhere, in botlib)
- `VectorLength` — defined in math utility (botlib/game shared)
- `Log_Print` — defined in `l_log.c`

# code/bspc/aas_cfg.h
## File Purpose
Defines the AAS (Area Awareness System) configuration structure used by the BSPC (BSP Compiler) tool. It encapsulates all physics simulation parameters and reachability scoring constants needed to classify navigation areas and build bot pathfinding data.

## Core Responsibilities
- Declare bounding box presence-type flags for grounded vs. airborne states
- Define `cfg_t`, the central configuration structure holding physics and reachability constants
- Expose a global `cfg` instance accessible across the BSPC tool
- Declare `DefaultCfg` and `LoadCfgFile` as the initialization entry points

## External Dependencies
- Relies on `aas_bbox_t` and `AAS_MAX_BBOXES` defined elsewhere (likely `aasfile.h` or `be_aas_def.h`)
- `vec3_t` from `q_shared.h` / mathlib
- `BBOXFL_GROUNDED` / `BBOXFL_NOTGROUNDED` flags used by presence-type logic in `aas_create.c` or equivalent

# code/bspc/aas_create.c
## File Purpose
Converts a BSP tree (produced by the BSPC map compiler) into a temporary AAS (Area Awareness System) world representation. It manages the full AAS creation pipeline from BSP leaf extraction through face classification, area merging, subdivision, and final file storage.

## Core Responsibilities
- Allocate and free temporary AAS data structures (faces, areas, nodes, node buffers)
- Convert BSP leaf nodes and their portals into convex AAS areas with classified faces
- Classify faces as ground, gap, solid, liquid, liquid-surface, or ladder
- Validate and repair face winding orientation relative to area centers
- Orchestrate the multi-pass AAS build pipeline in `AAS_Create`
- Assign area settings (flags, presence type, contents) from aggregated face flags
- Remove degenerate geometry (tiny faces, collinear winding points)

## External Dependencies
- `qbsp.h` — BSP types (`node_t`, `portal_t`, `plane_t`, `tree_t`), map globals (`mapplanes`, `entities`, `cancelconversion`, `freetree`, `source`), BSP pipeline functions
- `aasfile.h` — AAS file format constants (`FACE_*`, `AREA_*`, `AREACONTENTS_*`, `PRESENCE_*`)
- `aas_create.h` — Declarations for `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmp_aas_t`
- `aas_store.h` — `AAS_StoreFile`
- `aas_gsubdiv.h` — `AAS_GravitationalSubdivision`, `AAS_LadderSubdivision`
- `aas_facemerging.h` — `AAS_MergeAreaFaces`, `AAS_MergeAreaPlaneFaces`
- `aas_areamerging.h` — `AAS_MergeAreas`
- `aas_edgemelting.h` — `AAS_MeltAreaFaceWindings`
- `aas_prunenodes.h` — `AAS_PruneNodes`
- `aas_cfg.h` — `cfg` (physics config: `phys_gravitydirection`, `phys_maxsteepness`, `allpresencetypes`)
- `surfaceflags.h` — BSP content flags (`CONTENTS_SOLID`, `CONTENTS_WATER`, `CONTENTS_LADDER`, etc.)
- **Defined elsewhere:** `GetClearedMemory`, `FreeMemory`, `FreeWinding`, `ReverseWinding`, `CopyWinding`, `WindingCenter`, `WindingPlane`, `WindingArea`, `RemoveColinearPoints`, `WindingError`, `Log_Print`, `Log_Write`, `qprintf`, `Error`, `I_FloatTime`, `ThreadSetDefault`, `DotProduct`, `VectorCopy`, `VectorInverse`, `VectorScale`, `VectorAdd`

# code/bspc/aas_create.h
## File Purpose
Defines the temporary in-memory data structures used during AAS (Area Awareness System) world construction, along with the public interface for creating and manipulating those structures. It serves as the shared type contract between the BSPC tool's BSP-to-AAS conversion pipeline stages.

## Core Responsibilities
- Declare the `tmp_face_t`, `tmp_area_t`, `tmp_areasettings_t`, `tmp_node_t`, `tmp_nodebuf_t`, and `tmp_aas_t` intermediate structs used during AAS compilation
- Expose the top-level `AAS_Create` entry point for converting a loaded BSP map into an `.AAS` file
- Provide allocator/free declarations for all temporary AAS primitives
- Declare face-to-area linkage and removal helpers
- Declare geometry query helpers (`AAS_GapFace`, `AAS_GroundFace`, `AAS_FlipAreaFaces`, `AAS_CheckArea`)

## External Dependencies
- `winding_t` — polygon winding type; defined in `l_poly.h` / `qbsp.h` (not in this file)
- `AREA_PORTAL` (`1`) — flag constant used by face/area classification logic elsewhere in `bspc/`
- `NODEBUF_SIZE` (`128`) — compile-time slab size for node buffer

# code/bspc/aas_edgemelting.c
## File Purpose
Implements the "edge melting" pass for AAS (Area Awareness System) world generation, which refines face winding geometry by inserting shared boundary vertices between adjacent faces within the same area. This is a preprocessing step in BSP-to-AAS conversion that improves topological accuracy of convex area boundaries.

## Core Responsibilities
- For each pair of faces in an AAS area, detect vertices of one face that lie on the boundary edge of another face's winding
- Insert those detected vertices into the target winding via `AddWindingPoint`, splitting edges where needed
- Accumulate and report a count of total winding edge splits across all areas
- Log progress to both console (`qprintf`) and log file (`Log_Write`)

## External Dependencies
- `qbsp.h` — `plane_t`, `mapplanes[]`, `winding_t`, `qprintf`, logging utilities
- `../botlib/aasfile.h` — AAS data structure constants and types
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_aas_t`, `tmpaasworld`
- **Defined elsewhere:** `PointOnWinding`, `AddWindingPoint`, `FreeWinding` (winding geometry utilities, likely `l_poly.c`); `Log_Write` (`l_log.c`); `Error` (`l_cmd.c`)

# code/bspc/aas_edgemelting.h
## File Purpose
Header file declaring the public interface for the AAS edge-melting pass within the BSPC (BSP compiler) tool. It exposes a single function used to simplify area face geometry by merging redundant collinear edges in AAS area windings.

## Core Responsibilities
- Declares `AAS_MeltAreaFaceWindings`, the sole public entry point for the edge-melting subsystem
- Acts as the module boundary between the edge-melting implementation (`aas_edgemelting.c`) and the rest of the BSPC pipeline

## External Dependencies
- No includes in this header; the implementation (`aas_edgemelting.c`) depends on shared BSPC AAS internal structures (areas, faces, windings) defined elsewhere in the `bspc/` subsystem.
- `AAS_MeltAreaFaceWindings` — defined in `code/bspc/aas_edgemelting.c`

# code/bspc/aas_facemerging.c
## File Purpose
Implements face-merging passes over the temporary AAS world during BSP-to-AAS conversion. It reduces face count by coalescing coplanar, compatible faces within and across areas, simplifying the final AAS geometry.

## Core Responsibilities
- Attempt to merge two individual `tmp_face_t` windings into one (`AAS_TryMergeFaces`)
- Iterate all areas, retrying merges until no more are possible (`AAS_MergeAreaFaces`)
- Merge all same-plane faces within a single area unconditionally (`AAS_MergePlaneFaces`)
- Guard plane-face merges with a compatibility pre-check (`AAS_CanMergePlaneFaces`)
- Drive a full pass of per-plane face merging over all areas (`AAS_MergeAreaPlaneFaces`)
- Clean up consumed faces: remove from area lists and free them

## External Dependencies
- **`qbsp.h`** — pulls in `mapplanes[]`, `winding_t`, `plane_t`, `Log_Write`, `qprintf`, `MergeWindings`, `TryMergeWinding`, `FreeWinding`
- **`../botlib/aasfile.h`** — AAS file format constants (face flags, area flags, travel types); types not directly used in this file but needed by the broader AAS creation pipeline
- **`aas_create.h`** — `tmp_face_t`, `tmp_area_t`, `tmp_aas_t`, `tmpaasworld`, `AAS_RemoveFaceFromArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`
- **Defined elsewhere:** `tmpaasworld` (aas_create.c), `mapplanes` (map.c), `MergeWindings`/`TryMergeWinding`/`FreeWinding` (l_poly.c)

# code/bspc/aas_facemerging.h
## File Purpose
Public interface header for the AAS face merging subsystem within the BSPC (BSP Compiler) tool. Declares two functions responsible for merging coplanar faces within AAS areas to reduce geometry complexity during AAS file generation.

## Core Responsibilities
- Exposes the face merging API to other BSPC compilation units
- Declares area-level face merging (all faces across areas)
- Declares plane-constrained face merging (merging faces sharing the same plane)

## External Dependencies
- No includes in this header
- Both declared functions are **defined in** `code/bspc/aas_facemerging.c`
- Implicitly depends on global AAS world state structures defined in `aas_create.h` / `aas_store.h` (not visible here)

# code/bspc/aas_file.c
## File Purpose
Handles serialization and deserialization of AAS (Area Awareness System) files for the BSPC tool. It reads and writes the binary AAS navigation mesh format used by the Quake III bot system, including endian-swapping for cross-platform compatibility.

## Core Responsibilities
- Load AAS files from disk into the global `aasworld` structure, lump by lump
- Write AAS data from `aasworld` back to disk in the binary lump format
- Perform little-endian byte-swapping on all AAS data fields
- Apply a lightweight XOR obfuscation pass (`AAS_DData`) to the header on write/read
- Validate file identity (`AASID`) and version (`AASVERSION` / `AASVERSION_OLD`)
- Log reachability counts by travel type and AAS world totals

## External Dependencies
- `qbsp.h` — BSP tool types, `Error`, memory utilities (`GetClearedMemory`, `FreeMemory`), `LittleLong`, `LittleFloat`, `LittleShort`
- `botlib/aasfile.h` — All AAS data structure definitions, lump constants, travel type constants, `AASID`, `AASVERSION`
- `aas_store.h` — Declares `extern aas_t aasworld` (global AAS world state)
- `aas_create.h` — Included for `tmp_aas_t` context; not directly called here
- `Log_Print` — Defined elsewhere (logging subsystem)
- `aasworld` — Defined in `aas_store`; all data arrays populated/read here

# code/bspc/aas_file.h
## File Purpose
Public interface header for AAS (Area Awareness System) file I/O operations within the BSPC (BSP Compiler) tool. It exposes exactly two functions for writing and loading compiled AAS navigation data to and from disk.

## Core Responsibilities
- Declare the AAS file write entry point for serializing compiled navigation data
- Declare the AAS file load entry point for deserializing navigation data with optional subrange support

## External Dependencies
- `qboolean` — defined in `q_shared.h` (or equivalent Quake shared header); used as the boolean return type.
- Implementation resides in `code/bspc/aas_file.c`.
- AAS world model state populated/consumed by this module is defined elsewhere (`aas_store.h`, `aas_create.h`).

# code/bspc/aas_gsubdiv.c
## File Purpose
Implements gravitational and ladder-based geometric subdivision of temporary AAS areas during the BSPC map compilation process. It splits AAS areas along planes to ensure areas do not contain mixed ground/gap or ground/ladder regions that would confuse bot navigation.

## Core Responsibilities
- Split individual `tmp_face_t` polygons along a plane, producing front/back fragments
- Construct a split winding that clips the splitting plane to the convex bounds of an area
- Evaluate candidate split planes for quality (minimizing face splits, avoiding epsilon slivers)
- Find the best vertical split plane between ground/gap face pairs in an area
- Recursively subdivide the AAS BSP tree via gravitational subdivision (ground vs. gap separation)
- Recursively subdivide areas containing both ladder faces and ground faces via a horizontal plane through the lowest ladder vertex
- Patch the global BSP tree after ladder subdivisions to keep it consistent

## External Dependencies
- `qbsp.h` — `plane_t`, `mapplanes[]`, `FindFloatPlane`, `WindingIsTiny`, winding/polygon primitives
- `aasfile.h` — face flags (`FACE_GROUND`, `FACE_LADDER`, `FACE_GAP`), area content flags, presence types
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmpaasworld`, alloc/free/check helpers, `AAS_GapFace`
- `aas_store.h` — indirectly pulls in `aas_t aasworld`
- `aas_cfg.h` — `cfg` global (gravity direction via `cfg.phys_gravitydirection`)
- `FindPlaneSeperatingWindings` — defined elsewhere (polygon/geometry library)
- `ClipWindingEpsilon`, `BaseWindingForPlane`, `ChopWindingInPlace`, `FreeWinding` — winding library (defined elsewhere)
- `Log_Write`, `Log_Print`, `qprintf`, `Error` — logging/error utilities defined elsewhere

# code/bspc/aas_gsubdiv.h
## File Purpose
Declares two functions responsible for geometrically subdividing AAS (Area Awareness System) areas based on movement physics properties. This header is part of the BSPC (BSP Compiler) tool that converts BSP map data into AAS navigation data for bot pathfinding.

## Core Responsibilities
- Expose the gravitational subdivision pass interface for AAS area generation
- Expose the ladder subdivision pass interface for AAS area generation

## External Dependencies
- No includes in this header.
- **Defined elsewhere:** `tmpaasworld` — the global temporary AAS world structure used across the BSPC AAS construction pipeline.

# code/bspc/aas_map.c
## File Purpose
Transforms raw map brushes from BSP entities into AAS-ready geometry during the BSPC (BSP Compiler) offline tool's map conversion process. It handles brush expansion for player bounding boxes, entity validation, content classification, and coordinate-space transformation for moving entities such as rotating doors.

## Core Responsibilities
- Compute signed distances from bounding-box (AABB or capsule) origin offsets relative to brush planes, used for Minkowski-sum expansion
- Expand each `mapbrush_t` outward by a player bounding box so pathfinding geometry accounts for player size
- Set `texinfo` flags on brush sides to control which sides act as BSP splitters
- Validate map entities for AAS relevance (world, func_wall, func_door, triggers, etc.)
- Resolve `trigger_always` activation chains recursively to determine if a rotating door is permanently open
- Transform brush planes into world-space for entities with an `origin` or rotation (func_door_rotating)
- Classify and normalize brush contents (SOLID, LADDER, CLUSTERPORTAL, TELEPORTER, JUMPPAD, MOVER, etc.)
- Duplicate and expand solid/ladder brushes once per configured bounding-box type (normal, crouch)

## External Dependencies
- `qbsp.h` — `mapbrush_t`, `side_t`, `plane_t`, `entity_t`, `mapplanes`, `mapbrushes`, `brushsides`, global counters, winding utilities, `AddBrushBevels`, `FindFloatPlane`
- `l_mem.h` — memory allocation
- `botlib/aasfile.h` — `aas_bbox_t`, presence type constants
- `aas_store.h` — `AAS_MAX_BBOXES`
- `aas_cfg.h` — `cfg_t`, `cfg` global
- `game/surfaceflags.h` — `CONTENTS_*`, `SURF_SKIP`
- **Defined elsewhere:** `ValueForKey`, `FloatForKey`, `GetVectorForKey`, `CreateRotationMatrix`, `RotatePoint`, `VectorMA`, `VectorInverse`, `DotProduct`, `BaseWindingForPlane`, `ChopWindingInPlace`, `FreeWinding`, `WindingsNonConvex`, `ClearBounds`, `AddPointToBounds`, `Log_Print`, `Warning`, `Error`, `memset`

# code/bspc/aas_map.h
## File Purpose
Public interface header for the AAS map brush creation module within the BSPC (BSP Compiler) tool. It exposes a single function used to convert BSP map brushes into AAS-compatible geometry.

## Core Responsibilities
- Declares the interface for converting `mapbrush_t` geometry into AAS brush data
- Acts as the include boundary between `aas_map.c` and other BSPC modules that need to create AAS map brushes

## External Dependencies
- `mapbrush_t` — defined in `qbsp.h` or `map.h` (BSPC map representation)
- `entity_t` — defined in BSP entity headers (BSPC)
- Implementation: `code/bspc/aas_map.c`

# code/bspc/aas_prunenodes.c
## File Purpose
This file implements a BSP tree pruning pass for the AAS (Area Awareness System) build tool (BSPC). It eliminates redundant internal BSP nodes where both children resolve to the same merged area, and collapses double-solid-leaf nodes, reducing AAS tree complexity before final file output.

## Core Responsibilities
- Recursively traverse the temporary AAS BSP node tree post-area-merge
- Detect and collapse internal nodes whose two children reference the same final area (after following merge chains)
- Detect and free double-solid-leaf nodes (both children NULL)
- Free redundant child nodes via `AAS_FreeTmpNode`
- Count and report the total number of pruned nodes

## External Dependencies
- `qbsp.h` — core BSPC types (`tmp_node_t`, etc. indirectly via `aas_create.h`), logging utilities
- `botlib/aasfile.h` — AAS file format constants and structures (included for type definitions)
- `aas_create.h` — defines `tmp_node_t`, `tmp_area_t`, `tmp_aas_t`, `tmpaasworld` global, and `AAS_FreeTmpNode`
- `Log_Write`, `Log_Print` — defined elsewhere in the BSPC logging layer (`l_log.c`)
- `AAS_FreeTmpNode` — defined in `aas_create.c`
- `tmpaasworld` — global `tmp_aas_t` instance defined in `aas_create.c`

# code/bspc/aas_prunenodes.h
## File Purpose
Public header for the AAS node pruning subsystem within the BSPC (BSP Compiler) tool. It declares a single entry point used during AAS (Area Awareness System) tree post-processing to remove unnecessary or degenerate nodes from the compiled BSP/AAS tree.

## Core Responsibilities
- Exposes `AAS_PruneNodes` for use by other BSPC compilation stages that need to invoke node pruning on the AAS tree.

## External Dependencies
- No includes in this header.
- `AAS_PruneNodes` is defined in `code/bspc/aas_prunenodes.c` (defined elsewhere).

# code/bspc/aas_store.c
## File Purpose
Converts the intermediate (temporary) AAS representation built during BSP compilation into the final packed `aas_t` world structure, deduplicating vertices, edges, planes, and faces via hash tables before the data is serialized to disk.

## Core Responsibilities
- Allocate and free all `aasworld` arrays sized to worst-case maximums derived from the tmp world
- Deduplicate and intern vertices, edges, and planes into contiguous arrays using hash chains
- Convert `tmp_face_t` windings into `aas_face_t` records (with edge index)
- Convert `tmp_area_t` nodes into `aas_area_t` records with bounds and centroid
- Recursively walk `tmp_node_t` tree and emit `aas_node_t` BSP nodes
- Copy bounding box configuration from `cfg` into `aasworld`
- Log allocation / deallocation totals

## External Dependencies
- `qbsp.h` — `plane_t`, `winding_t`, `mapplanes`, `nummapplanes`, math utilities
- `botlib/aasfile.h` — all `aas_*_t` struct definitions, face/area flag constants
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmpaasworld`
- `aas_cfg.h` — `cfg` (bounding box config)
- **Defined elsewhere:** `GetClearedMemory`, `FreeMemory`, `Log_Print`, `Log_Write`, `PrintMemorySize`, `qprintf`, `Error`, `Q_rint`, `VectorCopy/Add/Scale/Negate/Clear`, `AddPointToBounds`, `ClearBounds`, `ReverseWinding`, `FreeWinding`, `PlaneTypeForNormal`

# code/bspc/aas_store.h
## File Purpose
Header for the BSPC (BSP Compiler) tool's AAS storage subsystem, defining capacity limits for all AAS data arrays and declaring the public interface for allocating, freeing, and persisting the compiled AAS world data.

## Core Responsibilities
- Define compile-time maximum element counts for every AAS data structure (bboxes, vertexes, planes, edges, faces, areas, nodes, portals, clusters)
- Expose `aasworld` as the global AAS world state (type `aas_t`)
- Declare the interface to store a finalized AAS world to disk
- Declare helpers for plane lookup and bulk AAS memory management
- Guard against botlib internals being pulled in during BSPC compilation via `BSPCINCLUDE`

## External Dependencies
- `../game/be_aas.h` — travel flags, `aas_trace_t`, `aas_entityinfo_t`, `aas_clientmove_t`, and other AAS public types
- `../botlib/be_aas_def.h` — `aas_t` struct definition, all AAS sub-types (`aas_bbox_t`, `aas_area_t`, `aas_reachability_t`, etc.), routing structures
- `vec3_t`, `qboolean` — defined elsewhere in `q_shared.h`
- `aasworld` global — defined in `aas_store.c` (not visible here)

# code/bspc/aasfile.h
## File Purpose
Defines the binary file format for Area Awareness System (AAS) files used by the bot navigation system in Quake III Arena. It specifies all data structures, constants, and lump identifiers needed to read and write `.aas` files that describe navigable regions of a map.

## Core Responsibilities
- Define the AAS binary file layout (header + 14 lumps)
- Enumerate all travel types bots can use to traverse the world
- Define face flags, area contents flags, and area flags for navigation queries
- Declare geometry primitives (vertices, planes, edges, faces, areas, BSP nodes)
- Declare higher-level navigation constructs (reachabilities, area settings, portals, clusters)
- Provide presence type constants for standing vs. crouching bot movement

## External Dependencies
- `vec3_t` — defined in `q_shared.h` (used for all 3D vectors/vertices)
- No function prototypes; all symbols defined here are data structure and preprocessor constant definitions only.

# code/bspc/be_aas_bspc.c
## File Purpose
This file is a BSPC (BSP Compiler) build-context adapter that provides stub implementations and wrappers for the botlib AAS (Area Awareness System) import interface. It bridges the botlib's `botlib_import_t` function table to the collision model (`CM_*`) and logging systems used by the offline map compiler tool, replacing the live engine's implementations.

## Core Responsibilities
- Define and populate the `botlib_import_t` struct for use in the BSPC tool context (not the live game engine)
- Wrap `CM_*` collision functions (`CM_BoxTrace`, `CM_PointContents`, `CM_InlineModel`, etc.) for botlib consumption
- Provide no-op stubs for debug visualization functions (`AAS_DebugLine`, `AAS_ClearShownDebugLines`)
- Provide a minimal timing function (`Sys_MilliSeconds`) via `clock()`
- Redirect print/log output to stdout and the BSPC log file
- Provide stubs for `Com_Memset`, `Com_Memcpy`, and `COM_Compress` required by shared code
- Drive the full AAS reachability and cluster computation pipeline via `AAS_CalcReachAndClusters`

## External Dependencies
- `../game/q_shared.h` — shared math, types, trace structs
- `../bspc/l_log.h` — `Log_Print`, `Log_Write`
- `../bspc/l_qfiles.h` — `quakefile_t`
- `../botlib/l_memory.h` — `GetMemory`, `FreeMemory`
- `../qcommon/cm_public.h` — `CM_LoadMap`, `CM_BoxTrace`, `CM_PointContents`, `CM_InlineModel`, `CM_ModelBounds`, `CM_EntityString`
- `../botlib/be_aas_def.h` — `aasworld` global (defined elsewhere)
- `Error` — declared extern; defined in BSPC utility code
- `AAS_LoadBSPFile`, `AAS_InitSettings`, `AAS_InitAASLinkHeap`, `AAS_InitAASLinkedEntities`, `AAS_SetViewPortalsAsClusterPortals`, `AAS_InitReachability`, `AAS_ContinueInitReachability`, `AAS_InitClustering` — all defined in other botlib/AAS source files

# code/bspc/be_aas_bspc.h
## File Purpose
Header file for the BSPC (BSP Compiler) AAS integration layer. It declares the single entry point used by the BSPC tool to trigger full AAS (Area Awareness System) reachability and cluster computation from a compiled BSP map file.

## Core Responsibilities
- Exposes `AAS_CalcReachAndClusters` as the public interface between the BSPC compilation pipeline and the AAS subsystem.

## External Dependencies
- `struct quakefile_s` — defined elsewhere (likely `code/bspc/qfiles.h` or a shared BSP header); only forward-declared here.

# code/bspc/brushbsp.c
## File Purpose
Implements the core BSP tree construction algorithm for the BSPC tool, converting a flat list of brushes into a binary space partitioning tree. It handles brush allocation/deallocation, plane-side testing, split-plane selection heuristics, brush splitting, and multithreaded iterative tree building.

## Core Responsibilities
- Allocate, copy, free, and bound `bspbrush_t` and `node_t` structures
- Determine which side of a plane a brush or AABB lies on (`BoxOnPlaneSide`, `TestBrushToPlanenum`)
- Select the best split plane from candidate brush sides using a cost heuristic (`SelectSplitSide`)
- Geometrically split a brush across a plane, producing two child brushes (`SplitBrush`)
- Partition a brush list into front/back children for a node (`SplitBrushList`)
- Build the BSP tree recursively (`BuildTree_r`) or iteratively with a thread-safe node queue/stack (`BuildTree`, `BuildTreeThread`)
- Classify leaf nodes with content flags and AAS-specific data (`LeafNode`)
- Track peak memory statistics for brushes, nodes, and windings

## External Dependencies
- **Includes:** `qbsp.h` (all BSP types and constants), `l_mem.h` (memory allocation), `../botlib/aasfile.h` (AAS format constants), `aas_store.h` (`aasworld`, AAS types), `aas_cfg.h` (`cfg` for AAS expansion bbox check), `<assert.h>`
- **Defined elsewhere:** `mapplanes[]`, `numthreads`, `drawflag`, `create_aas`, `cancelconversion`, `microvolume` (globals from `bspc.c`/`map.c`); winding utilities (`BaseWindingForPlane`, `ClipWindingEpsilon`, `ChopWindingInPlace`, `WindingArea`, `WindingMemory`); thread primitives (`ThreadLock`, `ThreadSemaphoreWait`, `AddThread`, `RemoveThread`); GL debug draw (`GLS_BeginScene`, `GLS_Winding`, `GLS_EndScene`); `Tree_Alloc`; `Log_Print`/`Log_Write`

# code/bspc/bspc.c
## File Purpose
This is the main entry point and global state hub for the BSPC (BSP Compiler) tool, a standalone offline utility that converts Quake BSP files into AAS (Area Awareness System) navigation files consumed by the bot AI. It parses command-line arguments and dispatches to the appropriate conversion pipeline.

## Core Responsibilities
- Define and own all global BSP/AAS compilation flags (nocsg, optimize, freetree, etc.)
- Parse `main()` command-line arguments and map them to compilation modes
- Dispatch to one of six compilation operations: BSP→MAP, BSP→AAS, reachability, clustering, AAS optimization, AAS info
- Construct output `.aas` file paths from input file metadata
- Enumerate all BSP files under a Quake directory tree (Win32 and POSIX)
- Collect and resolve argument file lists via glob/pak-aware `FindQuakeFiles`

## External Dependencies
- `qbsp.h` — BSP pipeline types, all map/brush/node/portal structs, and declarations for the majority of processing functions
- `l_mem.h` — memory allocation
- `botlib/aasfile.h` — AAS on-disk format constants and structs
- `botlib/be_aas_cluster.h` — `AAS_InitClustering`
- `botlib/be_aas_optimize.h` — `AAS_Optimize`
- `aas_create.h`, `aas_store.h`, `aas_file.h`, `aas_cfg.h` — AAS build/IO pipeline (defined elsewhere)
- `be_aas_bspc.h` — `AAS_CalcReachAndClusters`, `AAS_InitBotImport` (defined elsewhere)
- `use_nodequeue` — extern from `brushbsp.c`
- `calcgrapplereach` — extern from `be_aas_reach.c`
- POSIX: `unistd.h`, `glob.h`, `sys/stat.h`; Win32: `direct.h`, `windows.h`

# code/bspc/cfgq3.c
## File Purpose
This is a BSPC (BSP Compiler) configuration data file for Quake III Arena, written in a custom domain-specific script format (not standard C despite the `.c` extension). It defines the physical bounding volumes and movement physics parameters used by the AAS (Area Awareness System) generator when compiling bot navigation data from BSP maps.

## Core Responsibilities
- Define player bounding box dimensions for normal-stance and crouch-stance presence types
- Specify gravity, friction, and velocity physics constants for bot movement simulation
- Provide risk/reward cost weights (`rs_*`) for various movement actions (jumping, teleporting, grappling, etc.)
- Parameterize the AAS reachability analysis so the compiler can accurately model what movements are physically possible

## External Dependencies
- Parsed by the BSPC script/precompiler subsystem (`code/bspc/l_precomp.c`, `code/botlib/l_script.c`)
- Presence type macros (`PRESENCE_NONE`, `PRESENCE_NORMAL`, `PRESENCE_CROUCH`) mirror definitions in `code/botlib/be_aas_def.h`
- Physics values correspond to Quake III Arena engine constants (e.g., `g_gravity 800`, `pm_maxspeed 320`) defined in `code/game/bg_public.h` and `code/game/g_local.h`

# code/bspc/csg.c
## File Purpose
Implements Constructive Solid Geometry (CSG) operations for the BSPC (BSP Compiler) tool. It processes raw map brushes into non-overlapping convex brush sets suitable for BSP tree construction by performing boolean set operations (subtract, intersect, chop) on brush geometry.

## Core Responsibilities
- Validate brush convexity and bounds (`CheckBSPBrush`)
- Generate per-side windings for BSP brushes (`BSPBrushWindings`)
- Merge adjacent compatible brushes into single brushes (`TryMergeBrushes`, `MergeBrushes`)
- Subtract one brush volume from another, splitting as needed (`SubtractBrush`)
- Compute brush intersections and disjoint tests (`IntersectBrush`, `BrushesDisjoint`)
- Carve intersecting brushes into the minimum non-overlapping set (`ChopBrushes`)
- Build the initial brush list from map data, clipped to world bounds (`MakeBspBrushList`)
- Orchestrate the full world brush processing pipeline (`ProcessWorldBrushes`)

## External Dependencies
- `qbsp.h` — all core types (`bspbrush_t`, `mapbrush_t`, `side_t`, `plane_t`, `tree_t`, `node_t`) and shared globals
- `mapplanes[]`, `mapbrushes[]`, `map_mins`, `map_maxs` — defined in `map.c`
- `cancelconversion`, `nocsg`, `create_aas` — defined in `bspc.c`
- `WindingsNonConvex`, `BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding` — defined in winding/polygon utilities
- `AllocBrush`, `FreeBrush`, `CopyBrush`, `SplitBrush`, `BoundBrush`, `BrushBSP`, `CountBrushList`, `FreeBrushList`, `ResetBrushBSP` — defined in `brushbsp.c`
- `FindFloatPlane` — defined in `map.c`
- `AllocNode`, `Tree_Alloc` — defined in `tree.c` / `brushbsp.c`
- `Log_Print`, `Log_Write`, `qprintf` — logging utilities

# code/bspc/faces.c
## File Purpose
Implements BSP face construction for the BSPC tool, handling vertex deduplication, T-junction elimination, face merging, and face subdivision. It transforms raw portal windings into properly split and merged BSP faces ready for BSP file output.

## Core Responsibilities
- Deduplicate map vertices via spatial hashing or linear search (`GetVertexnum`)
- Emit and assign vertex indices to face windings (`EmitFaceVertexes`, `EmitVertexes_r`)
- Detect and fix T-junctions by splitting edges at intermediate vertices (`TestEdge`, `FixFaceEdges`, `FixEdges_r`)
- Merge coplanar, same-content, same-texinfo faces on the same node (`TryMerge`, `MergeNodeFaces`)
- Subdivide faces exceeding the surface-cache size limit (`SubdivideFace`, `SubdivideNodeFaces`)
- Allocate/free `face_t` objects and build them from BSP portals (`FaceFromPortal`)
- Track and emit shared BSP edges, preventing four-way edges (`GetEdge2`)

## External Dependencies
- **Includes:** `qbsp.h` (all BSP types, flags, map globals), `l_mem.h` (GetMemory/FreeMemory)
- **Defined elsewhere:** `dvertexes[]`, `numvertexes`, `dedges[]`, `numedges`, `texinfo[]`, `noweld`, `notjunc`, `nomerge`, `nosubdiv`, `noshare`, `subdivide_size` (from `bspc.c`/map globals); `TryMergeWinding`, `CopyWinding`, `ReverseWinding`, `FreeWinding`, `ClipWindingEpsilon` (from winding/poly library); `qprintf`, `Error` (from utility layer)

# code/bspc/gldraw.c
## File Purpose
Provides debug visualization utilities for the BSPC (BSP Compiler) tool, offering two distinct rendering paths: a local OpenGL window via the Windows `glaux` library, and a TCP socket-based remote GL server protocol. It is Windows-only and intended for offline BSP compilation debugging, not runtime game rendering.

## Core Responsibilities
- Initialize and clear a local OpenGL debug window using the `glaux` auxiliary library
- Set current draw color for subsequent winding renders (red, grey, black)
- Draw `winding_t` polygons as filled + outlined primitives in the local GL window
- Establish a TCP connection to a local GL server (`GLS_BeginScene`)
- Serialize and transmit `winding_t` geometry over a socket to the remote GL server
- Close the remote GL server connection (`GLS_EndScene`)

## External Dependencies
- `<windows.h>`, `<GL/gl.h>`, `<GL/glu.h>`, `<GL/glaux.h>` — Windows-only; `glaux` is a legacy auxiliary library
- `qbsp.h` — pulls in `winding_t`, `vec3_t`, `vec_t`, `qboolean`, and `Error()`
- Winsock (`WSAStartup`, `socket`, `connect`, `send`, `closesocket`) — defined in Windows SDK, linked externally
- `Error()` — defined elsewhere in BSPC (`l_cmd.c`)

# code/bspc/glfile.c
## File Purpose
Exports a BSP tree's portal geometry to a `.gl` text file for external GL-based visualization tools. It traverses the BSP tree recursively and writes visible portal windings with per-face shading data.

## Core Responsibilities
- Determine which sides of a portal are visible based on node contents
- Serialize winding point data (XYZ + greyscale lighting) to a `.gl` text file
- Traverse the BSP tree recursively, visiting only leaf nodes to emit portals
- Reverse winding order for back-facing portals
- Count and report the total number of GL faces written

## External Dependencies
- **`qbsp.h`** — pulls in all BSP types (`portal_t`, `node_t`, `tree_t`, `winding_t`, `plane_t`), `outbase` (global char array for output path prefix), and utility function declarations.
- **`ReverseWinding`** — defined in `l_poly.c` (via `qbsp.h` chain)
- **`FreeWinding`** — defined in `l_poly.c`
- **`Error`** — defined in `l_cmd.c`
- **`outbase`** — global `char[32]`, defined in `bspc.c`

# code/bspc/l_bsp_ent.c
## File Purpose
Parses and manages BSP map entity data for the BSPC (BSP Compiler) tool. It reads entity key-value pair lists from script tokens and provides accessor functions to query and mutate entity properties at compile time.

## Core Responsibilities
- Parse `{key value}` entity blocks from a script stream into `entity_t` structures
- Allocate and populate `epair_t` key-value pairs via the botlib script tokenizer
- Provide get/set accessors for entity key-value pairs (string, float, vector)
- Maintain a global flat array of all parsed map entities
- Strip trailing whitespace from parsed keys and values

## External Dependencies
- `l_cmd.h` — `copystring`, `Error`, `qboolean`
- `l_mem.h` — `GetMemory`, `FreeMemory`
- `l_math.h` — `vec_t`, `vec3_t`
- `l_log.h` — (included but not directly called in this file)
- `botlib/l_script.h` — `script_t`, `token_t`, `PS_ReadToken`, `PS_ExpectAnyToken`, `PS_UnreadLastToken`, `StripDoubleQuotes`
- `l_bsp_ent.h` — defines `entity_t`, `epair_t`, `MAX_MAP_ENTITIES` (defined elsewhere)
- `printf`, `sscanf`, `strlen`, `strcmp`, `memset`, `atof` — C standard library

# code/bspc/l_bsp_ent.h
## File Purpose
Declares the entity and key-value pair data structures used during BSP map parsing and AAS compilation in the BSPC tool. It provides the interface for reading, writing, and querying entity key-value properties parsed from Quake III BSP/map source files.

## Core Responsibilities
- Define the `epair_t` linked-list node for storing entity key-value string pairs
- Define the `entity_t` aggregate representing a parsed map entity (brushes, origin, portals, etc.)
- Expose the global entity array and entity count used during BSP processing
- Declare parsing functions for deserializing entities and epairs from a script token stream
- Declare accessors for typed key-value lookups (string, float, vector)
- Declare mutation function `SetKeyValue` for writing entity properties

## External Dependencies
- `vec3_t`, `vec_t`, `qboolean` — defined in `q_shared.h` / math headers
- `script_t` — defined in `l_script.h` (BSPC script/tokenizer subsystem)
- `MAX_MAP_ENTITIES` — guard-defined in this header (2048) if not already defined
- `StripTrailing`, `SetKeyValue`, etc. — implemented in `l_bsp_ent.c` (defined elsewhere)

# code/bspc/l_bsp_hl.c
## File Purpose
Implements loading, saving, and manipulation of Half-Life (GoldSrc) BSP format files for use in the BSPC (BSP compiler/converter) tool. It handles all lump types defined in the HL BSP format and bridges them into the BSPC entity parsing system.

## Core Responsibilities
- Allocate and free max-capacity buffers for all HL BSP lump types
- Load a Half-Life BSP file from disk into global arrays, with bounds checking
- Write a Half-Life BSP file from global arrays back to disk
- Perform little-endian byte swapping on all structured BSP lumps
- Compress and decompress visibility data (run-length encoding)
- Compute fast XOR-shift checksums for each loaded lump
- Parse and unparse the entity string lump using the botlib script system

## External Dependencies
- `l_cmd.h` — `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `Error`, `LittleLong`, `LittleShort`, `LittleFloat`, `qprintf`
- `l_math.h` — math types (`vec3_t`, etc.; not directly used in this file)
- `l_mem.h` — `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `botlib/l_script.h` — `LoadScriptMemory`, `SetScriptFlags`, `FreeScript`
- `l_bsp_hl.h` — all `hl_d*` struct type definitions and lump index constants (defined elsewhere)
- `l_bsp_ent.h` — `ParseEntity`, `entities[]`, `num_entities`, `epair_t` (defined elsewhere)

# code/bspc/l_bsp_hl.h
## File Purpose
Defines the on-disk BSP format structures and limits for Half-Life (GoldSrc engine) BSP version 30. It exposes extern declarations for all global BSP lump arrays and declares the utility API for loading, writing, and manipulating HL BSP files within the BSPC (BSP compiler/converter) tool.

## Core Responsibilities
- Define HL BSP v30 format constants (lump indices, version, map limits)
- Declare all BSP on-disk data structures (`hl_dmodel_t`, `hl_dnode_t`, `hl_dface_t`, etc.)
- Define leaf content type constants (`HL_CONTENTS_*`)
- Expose global extern arrays representing each loaded BSP lump
- Declare the HL BSP file I/O and utility API

## External Dependencies
- No explicit `#include` directives in this header; depends on surrounding build context to provide `byte`, `qboolean`, and basic C types.
- Implementation defined in `code/bspc/l_bsp_hl.c` (not shown here).
- `FastChecksum` is declared here but likely defined in a shared utility module.
- `hl_texinfo_t` reuses the same layout as Quake 2's texinfo (notable cross-format sharing).

# code/bspc/l_bsp_q1.c
## File Purpose
Implements loading, saving, and manipulation of Quake 1 BSP (Binary Space Partition) map files within the BSPC tool. It handles memory allocation for all BSP lumps, byte-swapping between little-endian disk format and host byte order, and serialization/deserialization of entity data.

## Core Responsibilities
- Allocate and free max-capacity BSP data arrays for all Q1 lump types
- Load a Q1 BSP file from disk, copying each lump into pre-allocated global arrays
- Write a Q1 BSP file to disk, serializing all lump arrays with proper byte ordering
- Byte-swap all BSP lump data between disk (little-endian) and host formats
- Parse the entity string lump into an in-memory `entities[]` array
- Serialize the in-memory entity array back into the entity string lump

## External Dependencies
- `l_cmd.h` — `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `Error`, `LittleLong`, `LittleFloat`, `LittleShort`, `qboolean`, `byte`
- `l_mem.h` — `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `botlib/l_script.h` — `script_t`, `LoadScriptMemory`, `SetScriptFlags`, `FreeScript`
- `l_bsp_q1.h` — all `q1_d*` type definitions, lump index constants, version constant (defined elsewhere)
- `l_bsp_ent.h` — `entities[]`, `num_entities`, `epair_t`, `ParseEntity` (defined elsewhere)

# code/bspc/l_bsp_q1.h
## File Purpose
Defines the on-disk data structures, lump layout constants, capacity limits, and extern declarations for Quake 1 BSP (version 29) files. It serves as the Q1 BSP format interface used by the BSPC tool to load, inspect, and convert Q1 maps into AAS data.

## Core Responsibilities
- Define Q1 BSP v29 format limits (map capacity constants)
- Declare all on-disk BSP lump structs (`q1_dnode_t`, `q1_dface_t`, `q1_dleaf_t`, etc.)
- Define lump index constants and the file header layout
- Expose global arrays holding the parsed BSP data to translation units
- Declare the BSP I/O and entity parsing API functions

## External Dependencies
- No standard library includes directly; relies on types (`byte`, `int`, `short`) from surrounding BSPC/Q3 shared headers
- All function bodies defined in `code/bspc/l_bsp_q1.c` (not shown)
- Guarded by `#ifndef QUAKE_GAME` — the extern declarations and function prototypes are excluded when building as an in-engine component

# code/bspc/l_bsp_q2.c
## File Purpose
Implements loading, saving, and manipulation of Quake II BSP files for the BSPC tool. It manages all BSP lump data (geometry, visibility, entities, brushes, etc.) in global arrays, and provides utility functions for geometry queries against the loaded BSP data.

## Core Responsibilities
- Allocate and free all Q2 BSP lump arrays at maximum capacity (`Q2_AllocMaxBSP` / `Q2_FreeMaxBSP`)
- Load a Q2 BSP file from disk into global arrays, with byte-swapping and texture fixup (`Q2_LoadBSPFile`)
- Write in-memory BSP data back to disk (`Q2_WriteBSPFile`)
- Perform endian byte-swapping on all BSP lumps (`Q2_SwapBSPFile`)
- Compress and decompress PVS visibility data (`Q2_CompressVis` / `Q2_DecompressVis`)
- Fix broken brush texture references by matching brush sides to rendered faces (`Q2_FixTextureReferences`)
- Parse/unparse entity string data to/from the `entities[]` array
- Provide winding/face geometry predicates (`InsideWinding`, `InsideFace`, `Q2_FaceOnWinding`, `Q2_BrushSideWinding`)

## External Dependencies
- `l_cmd.h` — `Error`, `LoadFile`, `SafeWrite`, `SafeOpenWrite`, `LittleLong/Short/Float`, `StripTrailing`
- `l_mem.h` — `GetClearedMemory`, `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `l_poly.h` — `winding_t`, `BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding`, `WindingArea`, `FreeWinding`, `WindingError`, `WindingIsTiny`
- `l_math.h` — `VectorSubtract`, `CrossProduct`, `VectorNormalize`, `DotProduct`, `VectorNegate`
- `../botlib/l_script.h` — `LoadScriptMemory`, `SetScriptFlags`, `ParseEntity`, `FreeScript`
- `q2files.h` — all Q2 BSP lump type definitions and constants (`LUMP_*`, `MAX_MAP_*`, `IDBSPHEADER`, `BSPVERSION`)
- `l_bsp_ent.h` — `num_entities`, `entities[]`, `ParseEntity` (defined elsewhere)
- `WindingIsTiny` — declared extern, defined elsewhere

# code/bspc/l_bsp_q2.h
## File Purpose
Public header for the Quake II BSP file interface used by the BSPC (BSP Compiler) tool. It declares all globally shared BSP lump arrays and their associated count variables, along with the functions needed to load, write, and manipulate Q2 BSP data.

## Core Responsibilities
- Declare all extern BSP lump data arrays (geometry, visibility, lighting, entities, etc.)
- Expose Q2 BSP file I/O functions (`Load`, `Write`, `Print`)
- Expose visibility compression/decompression routines
- Expose entity string parse/unparse utilities
- Provide memory management entry points (`AllocMaxBSP`, `FreeMaxBSP`)

## External Dependencies
- BSP lump types (`dmodel_t`, `dleaf_t`, `dplane_t`, `dnode_t`, `dface_t`, `dedge_t`, `dbrush_t`, `dbrushside_t`, `darea_t`, `dareaportal_t`, `dvis_t`, `texinfo_t`, `dvertex_t`, `dedge_t`) — defined elsewhere, likely `aasfile.h` / `q3files.h` / Q2 BSP format headers
- `MAX_MAP_*` constants — defined elsewhere (BSP format limits header)
- `byte` typedef — defined elsewhere (likely `q_shared.h`)

# code/bspc/l_bsp_q3.c
## File Purpose
Implements loading, parsing, writing, and preprocessing of Quake III Arena BSP files for the BSPC (BSP compiler/converter) tool. It manages all global BSP lump data and performs visible brush side detection needed for AAS area generation.

## Core Responsibilities
- Load a Q3 BSP file from disk into global lump arrays, handling byte-swapping for endianness
- Write modified BSP lump data back to disk
- Allocate and free all global BSP data arrays (`Q3_FreeMaxBSP`)
- Compute per-surface planes for planar draw surfaces (`Q3_CreatePlanarSurfacePlanes`)
- Determine which brush sides are "visible" (face-matched) vs. internal (`Q3_FindVisibleBrushSides`)
- Parse and unparse entity key/value strings from the entity lump
- Print BSP lump statistics for diagnostics

## External Dependencies
- `l_cmd.h` — `Error`, `SafeOpenWrite`, `SafeWrite`, `LittleLong`, `LittleFloat`, `qboolean`, `byte`
- `l_math.h` — `vec3_t`, `VectorSubtract`, `CrossProduct`, `VectorNormalize`, `DotProduct`, `VectorLength`
- `l_mem.h` — `GetMemory`, `GetClearedMemory`, `FreeMemory`
- `l_log.h` — `Log_Print`, `Log_Write`
- `l_poly.h` — `winding_t`, `BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding`, `FreeWinding`, `WindingArea`, `WindingError`, `WindingIsTiny`
- `l_bsp_q3.h` — Q3 BSP struct/constant definitions (defined elsewhere)
- `l_bsp_ent.h` — `entities`, `num_entities`, `epair_t`, `ParseEntity` (defined elsewhere)
- `../botlib/l_script.h` — `LoadScriptMemory`, `SetScriptFlags`, `FreeScript` (defined elsewhere)
- `l_qfiles.h` — `LoadQuakeFile`, `quakefile_s` (defined elsewhere)
- `forcesidesvisible` — `extern qboolean`, defined elsewhere in BSPC

# code/bspc/l_bsp_q3.h
## File Purpose
This header declares the global BSP data arrays and counts for a loaded Quake III Arena `.bsp` file, as well as the three public functions used to load, free, and parse that data. It serves as the interface between the BSPC tool's Q3-format BSP reader (`l_bsp_q3.c`) and the rest of the BSPC compiler pipeline.

## Core Responsibilities
- Expose all Q3 BSP lump data arrays as `extern` globals for cross-translation-unit access
- Expose corresponding element-count integers for each lump array
- Declare the three entry-point functions: load, free, and entity-parse

## External Dependencies
- **`q3files.h`** — defines all `q3_d*_t` struct types, lump constants (`Q3_LUMP_*`), and map size limits (`Q3_MAX_MAP_*`).
- **`struct quakefile_s`** — used by `Q3_LoadBSPFile`; defined elsewhere in the BSPC codebase (not in this header).
- **`byte`**, **`vec3_t`** — primitive typedefs defined elsewhere (likely `qfiles.h` / `q_shared.h`).
- **`surfaceflags.h`** — commented out; surface/content flag bit definitions not directly pulled in here.

# code/bspc/l_bsp_sin.c
## File Purpose
Implements BSP file I/O and in-memory storage for the SiN game engine BSP format within the BSPC tool. It handles loading, writing, byte-swapping, and memory management for all SiN BSP lumps, and includes geometry helpers for texture reference fixing.

## Core Responsibilities
- Allocate and free max-capacity BSP data arrays (`Sin_AllocMaxBSP` / `Sin_FreeMaxBSP`)
- Load a SiN BSP file from disk into global arrays (`Sin_LoadBSPFile`)
- Write global BSP arrays back to disk (`Sin_WriteBSPFile`)
- Byte-swap all BSP data between little-endian disk format and host format (`Sin_SwapBSPFile`)
- Compress and decompress PVS visibility data (`Sin_CompressVis` / `Sin_DecompressVis`)
- Parse and unparse entity key-value pairs from the entity lump string (`Sin_ParseEntities` / `Sin_UnparseEntities`)
- Fix brush-side texture references by matching faces to brush sides geometrically (`Sin_FixTextureReferences`)

## External Dependencies
- `l_cmd.h` — `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `Error`, `LittleLong`, `LittleShort`, `LittleFloat`, `LittleUnsigned*`, `StripTrailing`
- `l_mem.h` — `GetClearedMemory`, `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `l_poly.h` — `winding_t`, `CopyWinding`, `FreeWinding`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingArea`, `WindingError`, `WindingIsTiny` (declared extern in file)
- `l_bsp_ent.h` — `entities[]`, `num_entities`, `epair_t`, `entity_t`, `ParseEntity`
- `l_bsp_sin.h` — all `sin_d*` struct type definitions, lump constants (`SIN_LUMP_*`, `SIN_MAX_MAP_*`), surface flags
- `../botlib/l_script.h` — `LoadScriptMemory`, `SetScriptFlags`, `FreeScript`
- `WindingIsTiny` — declared forward in this file, defined elsewhere

# code/bspc/l_bsp_sin.h
## File Purpose
Public interface header for loading, writing, and manipulating Sin (Ritual Entertainment) BSP map files within the BSPC (BSP Compiler) tool. It declares all global BSP lump arrays and the API functions used to process Sin-format `.bsp` files for AAS (Area Awareness System) generation.

## Core Responsibilities
- Declare BSP format magic numbers/version constants for both Sin (`IBSP v41`) and SinGame (`RBSP v1`) variants
- Expose all parsed Sin BSP lump data as `extern` global arrays accessible across translation units
- Declare functions for loading, writing, printing, and vis-compressing Sin BSP data
- Declare entity string parse/unparse utilities for the Sin BSP entity lump

## External Dependencies
- `sinfiles.h` — defines all `sin_d*` struct types, lump index constants (`SIN_LUMP_*`), surface flags (`SURF_*`), `SINHEADER_LUMPS`, and map size limits (`SIN_MAX_MAP_*`)
- `byte`, `vec3_t` — defined elsewhere in shared headers (e.g., `qfiles.h` or `l_utils.h`)
- All function bodies defined in `l_bsp_sin.c` (not present here)

# code/bspc/l_cmd.c
## File Purpose
A general-purpose command-line and file utility library for the BSPC (BSP Compiler) tool. It provides portable OS abstraction for file I/O, path manipulation, string utilities, argument parsing, byte-order swapping, and CRC computation used throughout the BSP compilation pipeline.

## Core Responsibilities
- Fatal error and warning reporting (console and optional Win32 message box variants)
- File I/O wrappers with error-checked reads/writes and full-file load/save helpers
- Path string manipulation: extraction, extension handling, directory creation
- Command-line argument parsing and wildcard expansion (Win32 only)
- Byte-order (endianness) conversion for short, int, and float primitives
- CCITT CRC-16 computation
- Token parsing from C-string buffers (`COM_Parse`)
- Quake directory resolution from a given path (`SetQdirFromPath`)

## External Dependencies
- **Includes:** `l_cmd.h`, `l_log.h`, `l_mem.h`, `<sys/types.h>`, `<sys/stat.h>`, `<direct.h>` (Win32) or `<unistd.h>` (POSIX), `<windows.h>` (WINBSPC), `"io.h"` (Win32 wildcard expansion)
- **Defined elsewhere:** `Log_Write`, `Log_Close`, `Log_Print` (in `l_log.c`); `GetMemory`, `FreeMemory` (in `l_mem.c`); `WinBSPCPrint` (in WINBSPC platform layer)

# code/bspc/l_cmd.h
## File Purpose
A utility header for the BSPC (BSP Compiler) tool providing common command-line, file I/O, path manipulation, byte-order conversion, and string utility declarations. It mirrors the pattern of `cmdlib.h` found in other id Software tool codebases.

## Core Responsibilities
- Declare string utility functions (case-insensitive comparison, upper/lower conversion)
- Declare file I/O helpers (safe open/read/write, file loading, path operations)
- Declare byte-order swapping functions (Big/Little endian conversions)
- Declare argument/command-line parsing utilities
- Define shared global state for paths, archive mode, verbosity, and token parsing
- Provide the `qboolean`/`byte` typedefs and a portable `offsetof` macro
- Declare CRC checksum helpers

## External Dependencies
- Standard C library: `<stdio.h>`, `<string.h>`, `<stdlib.h>`, `<errno.h>`, `<ctype.h>`, `<time.h>`, `<stdarg.h>`
- All declared functions are **defined elsewhere** (in `l_cmd.c` / `cmdlib.c` within the BSPC tool)
- The `SIN` macro guard enables additional unsigned endian variants originally added for the SiN game engine codebase

# code/bspc/l_log.c
## File Purpose
Provides a simple file-based logging facility for the BSPC (BSP Compiler) tool. It manages a single global log file with functions to open, close, print, and flush log output, with optional console mirroring controlled by a `verbose` flag.

## Core Responsibilities
- Open and close a single global log file by filename
- Write formatted messages to the log file (with and without console mirroring)
- Normalize line endings to `\r\n` (CRLF) before writing to file
- Flush the log file on demand and after every write
- Provide access to the underlying `FILE*` handle for external use

## External Dependencies
- `<stdlib.h>`, `<stdio.h>`, `<string.h>` — standard C I/O and string utilities
- `qbsp.h` — pulls in `verbose` (extern global) and the `WinBSPCPrint` declaration (Windows GUI build only)
- `verbose` — extern boolean controlling console mirroring; defined elsewhere in BSPC
- `WinBSPCPrint` — defined elsewhere; only referenced under `WINBSPC` define

# code/bspc/l_log.h
## File Purpose
Public header for a simple logging utility used by the BSPC (BSP Compiler) tool. It declares the interface for opening, writing to, timestamping, and closing a log file, with an optional Windows-specific print hook.

## Core Responsibilities
- Declare the log file lifecycle API (open, close, shutdown)
- Declare formatted write functions (stdout+file, file-only, timestamped)
- Expose raw `FILE*` access for external consumers
- Provide a flush mechanism for the log file
- Conditionally declare a Windows GUI print callback (`WINBSPC`)

## External Dependencies
- `<stdio.h>` — `FILE*` type (implicitly required by `Log_FileStruct`; must be included before this header in translation units).
- `WINBSPC` — preprocessor symbol controlling the Windows GUI variant; defined elsewhere in the build system.
- Implementation defined in `code/bspc/l_log.c`.

# code/bspc/l_math.c
## File Purpose
Provides a general-purpose 3D math primitive library for the BSPC (BSP Compiler) tool. Implements vector, rotation, and bounding-box operations used throughout the BSP compilation pipeline. This is a standalone math utility layer, not connected to the runtime game engine.

## Core Responsibilities
- Euler angle decomposition into orthonormal basis vectors (forward/right/up)
- 3×3 rotation matrix concatenation and identity initialization
- Vector arithmetic: add, subtract, scale, dot product, cross product, MA, copy
- Vector normalization (in-place and out-of-place variants)
- Bounding-box management (clear, expand, radius)
- Color channel normalization to [0,1] range

## External Dependencies
- `l_cmd.h` — pulls in `<stdio.h>`, `<stdlib.h>`, `qboolean` typedef; provides general BSPC utility declarations.
- `l_math.h` — declares all types, macros, and function prototypes implemented here.
- `<math.h>` (via `l_math.h`) — `sin`, `cos`, `sqrt`, `fabs`, `floor`.
- `VectorLength`, `VectorClear`, `VectorScale`, `DotProduct` — defined/macro-expanded within this translation unit or its headers; no external linkage required.

# code/bspc/l_math.h
## File Purpose
Defines the core 3D math types, constants, and vector operation interfaces used throughout the BSPC (BSP Compiler) tool. It provides both macro-based inline operations and function declarations for vector arithmetic, normalization, bounds tracking, and rotation utilities.

## Core Responsibilities
- Declare scalar and vector typedefs (`vec_t`, `vec3_t`, `vec4_t`) with optional double-precision via `DOUBLEVEC_T`
- Provide inline macro implementations of common vector ops (dot product, add, subtract, scale, etc.)
- Declare function-form equivalents of vector ops for use where macros are inappropriate
- Declare geometric utilities: normalization, cross product, length, color normalization
- Declare spatial bounds management (`ClearBounds`, `AddPointToBounds`)
- Declare angle/rotation matrix utilities (`AngleVectors`, `R_ConcatRotations`, `CreateRotationMatrix`, `RotatePoint`)
- Define BSP-specific plane side constants (`SIDE_FRONT`, `SIDE_BACK`, `SIDE_ON`, `SIDE_CROSS`)

## External Dependencies
- `<math.h>` — for `M_PI`, trigonometric functions used by implementations
- `qboolean` — defined elsewhere in BSPC/shared headers (not defined here)
- Implementations: `l_math.c` (defined elsewhere)

# code/bspc/l_mem.c
## File Purpose
Provides memory allocation, tracking, and management for the BSPC (BSP Compiler) tool. Implements both a lightweight release mode and a debug mode (`MEMDEBUG`) with full block introspection, plus compatibility shims for Quake 3's `Hunk_*` and `Z_*` memory APIs.

## Core Responsibilities
- Allocate (`GetMemory`) and zero-initialize (`GetClearedMemory`) heap memory via `malloc`
- Track total allocated memory size via a global counter (`allocedmemory`) in release mode
- In debug mode, maintain a doubly-linked list of tagged `memoryblock_t` headers for leak detection and validation
- Provide `FreeMemory` with block validation (magic ID, pointer self-consistency) in debug mode
- Implement a simple linked-list hunk allocator (`Hunk_Alloc`, `Hunk_ClearHigh`) over `GetClearedMemory`
- Bridge Quake 3 engine `Z_Malloc`/`Z_Free` calls to the local allocator
- Log memory size summaries (`PrintMemorySize`, `PrintMemoryLabels`, `PrintUsedMemorySize`)

## External Dependencies
- **Includes:** `qbsp.h` (pulls in `malloc.h`, BSP types, and all bspc headers), `l_log.h`
- **External symbols used but not defined here:**
  - `Error` — fatal error handler (defined in `l_cmd.c`)
  - `Log_Print`, `Log_Write` — logging (defined in `l_log.c`)
  - `_msize` — Win32 CRT heap query (platform SDK)
  - `malloc`, `free`, `memset` — C standard library

# code/bspc/l_mem.h
## File Purpose
Public interface header for the BSPC tool's custom memory management subsystem. It declares allocation and deallocation routines with an optional debug mode that captures source location metadata at the call site via macros.

## Core Responsibilities
- Declare `GetMemory` / `GetClearedMemory` for raw and zero-initialized heap allocation
- Provide debug-mode macro overrides that inject label, file, and line information into every allocation call
- Declare `FreeMemory` as the single deallocation entry point
- Expose utility queries: per-block size (`MemorySize`), human-readable size printing (`PrintMemorySize`), and total allocated byte count (`TotalAllocatedMemory`)

## External Dependencies
- No includes within this header.
- **Defined elsewhere:** `GetMemory`, `GetClearedMemory`, `FreeMemory`, `MemorySize`, `PrintMemorySize`, `TotalAllocatedMemory`, and their debug counterparts — all implemented in `code/bspc/l_mem.c`.

# code/bspc/l_poly.c
## File Purpose
Implements convex polygon (winding) operations for the BSPC map compiler tool. Provides the full lifecycle—allocation, clipping, merging, validation, and geometry queries—for `winding_t` structures used during BSP construction and AAS area generation.

## Core Responsibilities
- Allocate and free `winding_t` objects with optional single-threaded usage statistics
- Generate a large base winding from a plane definition
- Clip windings against planes (epsilon-tolerant, in-place and copy variants)
- Merge two adjacent windings sharing a common edge (both convex-safe and brute-force variants)
- Query winding geometry: area, center, bounds, plane, side classification
- Validate winding integrity and report structured error codes
- Remove degenerate geometry: colinear points, duplicate/equal points

## External Dependencies
- `<malloc.h>` — system allocation (underlying `GetMemory`)
- `l_cmd.h` — `Error`, `qboolean`, `vec_t` primitives
- `l_math.h` — `vec3_t`, `DotProduct`, `CrossProduct`, `VectorNormalize`, `VectorLength`, etc.
- `l_log.h` — `Log_Print`, `Log_Write` for degenerate-case diagnostics
- `l_mem.h` — `GetMemory`, `FreeMemory`, `MemorySize` (custom allocator with size tracking)
- `numthreads` — extern from the BSPC threading system (defined elsewhere)
- `vec3_origin` — extern zero-vector (defined in `l_math.c`)

# code/bspc/l_poly.h
## File Purpose
Declares the interface for convex polygon (winding) operations used during BSP compilation. Windings represent convex polygons bounded by a set of 3D points and are the fundamental geometric primitive for CSG, clipping, and plane operations in the BSPC tool.

## Core Responsibilities
- Define the `winding_t` structure and associated limits/constants
- Declare allocation, deallocation, and memory-tracking functions for windings
- Declare geometric operations: area, center, bounds, plane extraction
- Declare clipping and chopping operations against planes
- Declare winding merging, reversing, and copying utilities
- Declare validation/error-checking functions for winding integrity
- Declare point-on-edge and plane-separation queries for BSP adjacency tests

## External Dependencies
- Implicit dependency on `vec3_t`, `vec_t` from `mathlib.h` / `q_shared.h` (defined elsewhere).
- Implementation lives in `code/bspc/l_poly.c` (defined elsewhere).
- `MAX_POINTS_ON_WINDING` (96) caps the polygon vertex count during clipping operations.

# code/bspc/l_qfiles.c
## File Purpose
Provides utilities for discovering and loading Quake engine game files across multiple archive formats (PAK, PK3/ZIP, loose files) on both Windows and POSIX platforms. It abstracts the differences between id-format PAK files, Sin PAK files, and ZIP-based PK3 packages into a unified `quakefile_t` linked-list interface.

## Core Responsibilities
- Map file extensions to Quake file type enum constants (bidirectional lookup)
- Implement glob/wildcard pattern matching (`FileFilter`, `StringContains`) for archive entry filtering
- Enumerate files inside ZIP/PK3 archives matching a filter
- Enumerate files inside id PAK and Sin PAK archives matching a filter
- Recursively search filesystem paths (Win32 `FindFirstFile`/POSIX `glob`) to locate loose files or pak containers
- Parse a composite filter string (`path/to/archive.pak/subpath/*.bsp`) and dispatch to the correct loader
- Load or partially read a matched `quakefile_t` into memory, handling both ZIP streams and raw file offsets

## External Dependencies
- `qbsp.h` (transitively pulls in all BSPC headers)
- `l_qfiles.h` — declares `quakefile_t`, `QFILETYPE_*`, `QFILEEXT_*` constants
- `unzip.h` / minizip — `unzFile`, `unz_s`, `unzOpen`, `unzGetGlobalInfo`, etc.
- `q2files.h` — `dpackheader_t`, `dpackfile_t`, `dsinpackfile_t`, `IDPAKHEADER`, `SINPAKHEADER`
- `l_cmd.h` / `l_utils.h` — `ExtractFileExtension`, `ConvertPath`, `AppendPathSeperator`, `SafeOpenRead`, `SafeRead`, `Q_filelength`, `GetMemory`, `Error`, `Warning`
- Win32: `<windows.h>`, `FindFirstFile`/`FindNextFile`, `_splitpath`, `_stat`
- POSIX: `<glob.h>`, `<unistd.h>`, `glob`/`globfree`, `stat`
- `LittleLong` — byte-order conversion, defined elsewhere

# code/bspc/l_qfiles.h
## File Purpose
Header defining the interface for locating, identifying, and loading Quake engine asset files across multiple container formats (PAK, PK3/ZIP, raw filesystem). It provides a unified abstraction over Quake 1/2/3/Sin pack file types for use by the BSPC BSP compiler tool.

## Core Responsibilities
- Define bitmask constants identifying Quake file types (BSP, MD2, MD3, AAS, WAL, etc.)
- Define canonical uppercase file extension strings for each type
- Declare the `quakefile_t` linked-list node representing a located asset file
- Declare the `dsinpackfile_t` structure for Sin pack directory entries
- Expose API for file-type detection by extension or filename
- Expose API for glob-style file searching across pack and filesystem sources
- Expose API for loading or partially reading a located Quake file into memory

## External Dependencies
- `../qcommon/unzip.h` — provides `unz_s` (embedded by value in `quakefile_t.zipinfo`) and the full minizip API used by the implementation (`l_qfiles.c`).
- `_MAX_PATH` — conditionally defined here as 1024 if not already provided by the platform.
- Implementation (`l_qfiles.c`) defined elsewhere; all function bodies external to this file.

# code/bspc/l_threads.c
## File Purpose
Provides a cross-platform threading abstraction layer for the BSPC (BSP compiler) tool. It implements mutexes, semaphores, work dispatch, and thread lifecycle management with four platform-specific backends: Win32, OSF1 (Digital Unix), Linux (pthreads), and IRIX (sproc), plus a no-op single-threaded fallback.

## Core Responsibilities
- Dispatch a fixed work queue across N worker threads with progress reporting
- Provide mutex (ThreadLock/ThreadUnlock) and semaphore primitives per platform
- Manage a linked list of dynamically spawned threads (AddThread/RemoveThread)
- Auto-detect CPU count for default thread count (Win32, IRIX)
- Gate all multi-threaded paths behind the `threaded` flag to catch misuse
- Provide `RunThreadsOnIndividual` as a higher-level wrapper that assigns one work item per thread invocation

## External Dependencies
- `l_cmd.h` — `Error`, `qprintf`, `I_FloatTime`, `qboolean`
- `l_threads.h` — declares all exported symbols
- `l_log.h` — `Log_Print`
- `l_mem.h` — `GetMemory`, `FreeMemory`
- **Win32:** `<windows.h>` — `CRITICAL_SECTION`, `CreateThread`, `WaitForSingleObject`, `CreateSemaphore`, `ReleaseSemaphore`, `GetSystemInfo`
- **OSF1/Linux:** `<pthread.h>` — `pthread_create`, `pthread_join`, `pthread_mutex_*`
- **Linux:** `<semaphore.h>` — `sem_init`, `sem_wait`, `sem_post`, `sem_destroy`
- **IRIX:** `<task.h>`, `<abi_mutex.h>`, `<sys/prctl.h>` — `sprocsp`, `spin_lock`, `release_lock`, `init_lock`

# code/bspc/l_threads.h
## File Purpose
Public header declaring the threading API for the BSPC (BSP compiler) tool. It exposes thread management, work dispatch, mutual exclusion (mutex), and semaphore primitives used during parallel BSP/AAS compilation tasks.

## Core Responsibilities
- Declare the global thread count variable (`numthreads`)
- Expose thread pool initialization and work-queue dispatch functions
- Declare mutex lock/unlock primitives for critical section protection
- Declare semaphore primitives for producer/consumer synchronization
- Declare dynamic thread add/remove and join-all utilities

## External Dependencies
- `qboolean` — defined in shared Q3 headers (e.g., `q_shared.h`)
- Implementations defined in `code/bspc/l_threads.c` (platform-specific: Win32, POSIX pthreads, or null/single-threaded stub)

# code/bspc/l_utils.c
## File Purpose
Provides cross-platform filesystem path utility functions for both the BSPC map compiler and the BOTLIB bot library. It normalizes path separators and ensures paths are properly terminated, with a conditionally compiled vector-to-angles conversion for BOTLIB use.

## Core Responsibilities
- Convert direction vectors to Euler angles (BOTLIB only)
- Normalize filesystem path separator characters to the platform-appropriate character
- Append a trailing path separator to directory strings safely
- Provide disabled (guarded with `#if 0`) legacy Quake 2 PAK file search routines

## External Dependencies
- **BOTLIB build path:** `q_shared.h`, `qfiles.h`, `botlib.h`, `l_log.h`, `l_libvar.h`, `l_memory.h`, `be_interface.h`
- **BSPC build path:** `qbsp.h`, `l_mem.h`
- **`PATHSEPERATOR_CHAR`** — defined elsewhere (platform headers or `qbsp.h`/`l_utils.h`)
- **`M_PI`, `atan2`, `sqrt`** — standard C `<math.h>`
- **`PITCH`, `YAW`, `ROLL`** — index constants from `q_shared.h`
- **`Log_Write`** — declared in `l_log.h`; used only in the disabled `#if 0` block
- **`LibVarGetString`** — declared in `l_libvar.h`; used only in the disabled block

# code/bspc/l_utils.h
## File Purpose
A utility header for the BSPC (BSP Compiler) tool, providing cross-platform path handling macros, math convenience macros, and declarations for file-finding utilities used during BSP/AAS compilation.

## Core Responsibilities
- Define cross-platform path separator macros (`\\` vs `/`)
- Provide math utility macros (random, clamp, abs, axis indices)
- Declare the `foundfile_t` structure for locating files inside pak archives
- Declare file-search functions for locating Quake assets on disk or in pak files
- Declare the `Vector2Angles` conversion utility

## External Dependencies
- `vec3_t` — defined in Quake shared math headers (`q_shared.h` or equivalent)
- `qboolean` — defined in `q_shared.h`
- `rand()` — standard C library (used in `random()` macro)
- `BOTLIB` — compile-time preprocessor flag controlling `FindQuakeFile` signature
- `MAX_PATH` — guarded; fallback defined here as 64 if not previously defined

# code/bspc/leakfile.c
## File Purpose
Generates a `.lin` leak trace file for the BSPC (BSP Compiler) tool. It traces the shortest portal path from the outside leaf to an occupied (entity-containing) leaf, enabling map editors like QE3 to visualize map leaks.

## Core Responsibilities
- Check whether the BSP tree's outside node is occupied (i.e., a leak exists)
- Traverse the portal graph greedily, following the path of decreasing `occupied` values
- Compute the center point of each portal winding along the path
- Write all trace points as XYZ coordinates to a `.lin` text file
- Append the final occupant entity's origin as the last point

## External Dependencies
- `qbsp.h` — aggregates all local BSPC headers; provides `tree_t`, `node_t`, `portal_t`, `winding_t`, entity types, and global declarations
- `WindingCenter` — defined in `l_poly.c` (via `l_poly.h`)
- `GetVectorForKey` — defined in `l_bsp_ent.c`
- `qprintf`, `Error` — defined in `l_cmd.c`
- `source` (global char array) — defined in `bspc.c`

# code/bspc/map.c
## File Purpose
This file is the central map data manager for the BSPC (BSP Compiler) tool. It handles plane management, brush geometry construction, and multi-format BSP map loading/writing. It serves as the unified interface for converting BSP files from various Quake-engine games into a normalized internal map representation.

## Core Responsibilities
- Maintain global arrays of map planes, brushes, and brush sides
- Find or create float planes with hash-based deduplication
- Add axial and edge bevel sides to brushes for AAS expansion
- Generate brush side windings and bounding boxes
- Write map data back to `.map` text files (multi-format aware)
- Dispatch BSP loading to the correct format handler (Q1/Q2/Q3/HL/SIN)
- Reset all map state between load operations

## External Dependencies
- `qbsp.h` — core types (`plane_t`, `mapbrush_t`, `side_t`, `winding_t`, `entity_t`, `face_t`), math macros, all subsystem prototypes
- `l_bsp_hl.h`, `l_bsp_q1.h`, `l_bsp_q2.h`, `l_bsp_q3.h`, `l_bsp_sin.h` — per-format BSP loader interfaces
- `l_mem.h` — `FreeMemory`, `FreeWinding`, `BaseWindingForPlane`, `ChopWindingInPlace`
- `aasfile.h` / `aas_store.h` / `aas_cfg.h` — AAS bounding box constants (included, not directly used in visible code)
- Defined elsewhere: `entities[]`, `num_entities`, `epair_t`, `entity_t`, `ReadQuakeFile`, `I_FloatTime`, `TextureAxisFromPlane`, `VectorNormalize2`, `Log_Write`, `Log_Print`, `Error`

# code/bspc/map_hl.c
## File Purpose
Converts Half-Life BSP files into the BSPC tool's internal map brush representation. It reconstructs solid geometry by recursively splitting a world-bounding brush along BSP node planes, then applies texture information and merges adjacent brushes before emitting final `mapbrush_t` entries.

## Core Responsibilities
- Load and parse a Half-Life BSP file and its entities
- Classify texture names to content types (solid, water, lava, slime)
- Recursively reconstruct `bspbrush_t` geometry from the HL BSP tree
- Assign texture/texinfo data to brush sides by matching face overlap
- Split brushes at face boundaries when multiple textures conflict on a side
- Merge compatible adjacent brushes to reduce brush count
- Convert `bspbrush_t` records into `mapbrush_t` entries, optionally invoking AAS brush creation

## External Dependencies
- **Includes:** `qbsp.h` (types, globals, brush/plane utilities), `l_bsp_hl.h` (HL BSP structures and loader), `aas_map.h` (`AAS_CreateMapBrushes`)
- **Defined elsewhere:** `hl_dleafs`, `hl_dnodes`, `hl_dplanes`, `hl_dmodels`, `hl_dfaces`, `hl_texinfo`, `hl_dtexdata`, `hl_dedges`, `hl_dvertexes`, `hl_dsurfedges`, `hl_numfaces`, `hl_texdatasize` (all from `l_bsp_hl`); `mapplanes`, `map_mins`, `map_maxs`, `map_texinfo`, `map_numtexinfo`, `nummapbrushes`, `nummapbrushsides`, `mapbrushes[]`, `brushsides[]`, `entities`, `num_entities` (all from `map.c`/`qbsp.h`); `lessbrushes`, `nobrushmerge`, `create_aas` (from `bspc.c`)

# code/bspc/map_q1.c
## File Purpose
Converts a Quake 1 (and Half-Life) BSP file into the BSPC tool's internal map brush representation. It reconstructs solid, water, slime, and lava brushes by recursively carving a bounding box with BSP node planes, then textures and optionally merges the result before forwarding to AAS or map export pipelines.

## Core Responsibilities
- Classify texture names to Q1 content types (solid, water, slime, lava)
- Recursively split a world-bounding brush along all BSP node planes to regenerate geometry (`Q1_CreateBrushes_r`)
- Assign texture info to brush sides by matching BSP faces that overlap each side's winding (`Q1_TextureBrushes`)
- Fix content-mismatched textures on liquid brushes (`Q1_FixContentsTextures`)
- Merge adjacent same-content brushes to reduce brush count (`Q1_MergeBrushes`)
- Convert internal `bspbrush_t` records into `mapbrush_t` entries for the global map arrays (`Q1_BSPBrushToMapBrush`)
- Orchestrate the full load pipeline per entity/model (`Q1_LoadMapFromBSP`)

## External Dependencies
- `qbsp.h` — all core BSP types (`bspbrush_t`, `mapbrush_t`, `side_t`, `plane_t`), global map arrays, and utility function declarations
- `l_bsp_q1.h` — Q1 BSP lump types and loaded BSP data arrays (`q1_dleafs`, `q1_dnodes`, `q1_dfaces`, `q1_dplanes`, etc.)
- `aas_map.h` — `AAS_CreateMapBrushes` (defined in `aas_map.c`)
- **Defined elsewhere:** `FindFloatPlane`, `BrushFromBounds`, `AllocBrush`, `FreeBrush`, `CopyBrush`, `SplitBrush`, `TryMergeBrushes`, `BoundBrush`, `BrushVolume`, `MakeBrushWindings`, `AddBrushBevels`, `CheckBSPBrush`, `BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding`, `WindingArea`, `FreeWinding`, `ClipWindingEpsilon`, `Log_Print`, `Q_strcasecmp`, `Q_strncasecmp`, `ValueForKey`, `qprintf`, `Error`, `map_texinfo[]`, `map_mins`/`map_maxs`, `lessbrushes`, `nobrushmerge`, `create_aas`, `nummapbrushes`, `nummapbrushsides`, `entities[]`, `num_entities`

# code/bspc/map_q2.c
## File Purpose
Handles loading and parsing of Quake 2 map data for the BSPC (BSP Compiler) tool, supporting both `.map` text format and BSP binary format. It converts Q2 brush/entity data into the internal `mapbrush_t`/`entity_t` representation used by the AAS generation pipeline.

## Core Responsibilities
- Parse Q2 `.map` text files into internal map structures (`Q2_LoadMapFile`)
- Load map geometry directly from compiled Q2 BSP files (`Q2_LoadMapFromBSP`)
- Determine brush contents/surface flags from texture info
- Handle special entity types: `func_group`, `func_areaportal`, origin brushes
- Convert BSP-format brushes (`dbrush_t`) to map brushes (`mapbrush_t`)
- Build per-brush model number mappings by traversing the BSP tree iteratively
- Populate `map_texinfo[]` from loaded Q2 texinfo data

## External Dependencies
- `qbsp.h` — all core types and globals (`mapbrushes`, `brushsides`, `mapplanes`, `entities`, etc.)
- `l_mem.h` — `GetMemory`, `FreeMemory`
- `botlib/aasfile.h` — `aas_bbox_t` (included for AAS type sizes)
- `aas_store.h` — `AAS_MAX_BBOXES`
- `aas_cfg.h` — `cfg_t` / AAS physics config
- `aas_map.h` — `AAS_CreateMapBrushes` (defined elsewhere)
- `l_bsp_q2.h` — `Q2_LoadBSPFile`, `Q2_ParseEntities`, BSP lump globals (`dbrushes`, `dbrushsides`, `dleafs`, `dleafbrushes`, `dfaces`, `texinfo`, etc.) — defined elsewhere

# code/bspc/map_q3.c
## File Purpose
Converts a compiled Quake III BSP file into the internal `mapbrush_t` representation used by the BSPC tool. It handles both solid brushes and curved patch surfaces, translating BSP-native structures into map geometry suitable for AAS generation or further BSP processing.

## Core Responsibilities
- Determine consolidated content flags for a map brush from its sides (`Q3_BrushContents`)
- Register BSP dplane entries into the map-plane table (`Q3_DPlanes2MapPlanes`)
- Convert a single `q3_dbrush_t` BSP brush into a `mapbrush_t` (`Q3_BSPBrushToMapBrush`)
- Parse all brushes belonging to a BSP model/entity (`Q3_ParseBSPBrushes`, `Q3_ParseBSPEntity`)
- Tessellate Q3 patch surfaces into convex brush proxies for collision/AAS (`AAS_CreateCurveBrushes`)
- Orchestrate full BSP-to-map load pipeline and compute world AABB (`Q3_LoadMapFromBSP`)
- Reset transient state between map loads (`Q3_ResetMapLoading`)

## External Dependencies

- **`qbsp.h`** — master include: map globals, `plane_t`, `mapbrush_t`, `side_t`, `entity_t`, utility prototypes.
- **`l_bsp_q3.h`** — Q3 BSP lump types (`q3_dbrush_t`, `q3_dbrushside_t`, `q3_dshaders`, `q3_drawSurfaces`, etc.) and `Q3_LoadBSPFile`, `Q3_ParseEntities`.
- **`cm_patch.h`** — `CM_GeneratePatchCollide`, `patchCollide_t`, `facet_t`.
- **`aas_map.h`** — `AAS_CreateMapBrushes`.
- **`surfaceflags.h`** — `SURF_HINT`, `SURF_SKIP`, `SURF_NODRAW`, `CONTENTS_*` constants.
- **Defined elsewhere:** `FindFloatPlane`, `MakeBrushWindings`, `MarkBrushBevels`, `AddBrushBevels`, `BrushExists`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `ClearBounds`, `AddPointToBounds`, `PrintContents`, `Log_Write`, `Log_Print`, `qprintf`, `Error`.

# code/bspc/map_sin.c
## File Purpose
Converts Sin game BSP data into the BSPC tool's internal map brush representation for AAS (Area Awareness System) generation. It handles both direct `.map` file parsing (commented out) and BSP-file-to-mapbrush conversion paths, including Sin-specific content/surface flag semantics.

## Core Responsibilities
- Determine brush content flags from side surface data (`Sin_BrushContents`)
- Initialize a default `map_texinfo` entry and copy Sin BSP texinfo data (`Sin_CreateMapTexinfo`)
- Traverse the BSP tree iteratively to assign model numbers to leaf brushes (`Sin_SetBrushModelNumbers` via node stack helpers)
- Convert individual `sin_dbrush_t` BSP brushes into `mapbrush_t` map brushes (`Sin_BSPBrushToMapBrush`)
- Parse all BSP brushes belonging to a given entity (`Sin_ParseBSPBrushes`, `Sin_ParseBSPEntity`)
- Drive the full BSP-to-map load pipeline (`Sin_LoadMapFromBSP`)
- Reset loader state between runs (`Sin_ResetMapLoading`)

## External Dependencies
- `qbsp.h` — all core types (`mapbrush_t`, `side_t`, `plane_t`, global arrays, constants)
- `l_bsp_sin.h` / `sinfiles.h` — Sin BSP data arrays (`sin_dbrushes`, `sin_texinfo`, `sin_dleafs`, etc.)
- `aas_map.h` — `AAS_CreateMapBrushes` (defined in `aas_map.c`)
- **Defined elsewhere:** `FindFloatPlane`, `MakeBrushWindings`, `MarkBrushBevels`, `BrushExists`, `AddBrushBevels`, `Log_Print`, `Sin_LoadBSPFile`, `Sin_ParseEntities`, `ValueForKey`, `GetVectorForKey`

# code/bspc/nodraw.c
## File Purpose
A null/stub implementation of the BSP compiler's OpenGL debug drawing interface. All functions are empty no-ops, serving as a build target for headless/server-side BSP compilation where no graphical debug visualization is needed.

## Core Responsibilities
- Provide link-time stubs for the GL scene visualization API declared in `qbsp.h` (gldraw.c section)
- Define the global drawing state variables `draw_mins`, `draw_maxs`, and `drawflag`
- Allow the BSPC tool to compile and link without a real GL debug renderer

## External Dependencies
- `qbsp.h` — pulls in all BSPC types (`winding_t`, `vec3_t`, `qboolean`, etc.) and the `extern` declarations for the symbols defined here
- `winding_t` — defined in `l_poly.h` (via `qbsp.h`); used only as a pointer parameter in stubs
- `GLSERV_PORT` (25001) — defined locally but never referenced; implies a GL server protocol exists elsewhere

# code/bspc/portals.c
## File Purpose
Implements BSP portal generation, entity flood-fill, and area classification for the BSPC (BSP Compiler) tool. Portals are convex polygon boundaries between adjacent BSP leaf nodes, used for PVS (Potentially Visible Set) computation and area portal detection.

## Core Responsibilities
- Allocate/free `portal_t` objects with memory tracking
- Build axis-aligned bounding portals for the BSP headnode (`MakeHeadnodePortals`)
- Create and split node portals during BSP tree traversal
- Flood-fill the BSP tree from entity origins to identify reachable vs. outside space
- Fill unreachable leaves as solid (`FillOutside`)
- Flood-classify leaves into numbered areas separated by `CONTENTS_AREAPORTAL` nodes
- Mark brush sides as visible when referenced by portals

## External Dependencies
- **`qbsp.h`**: All core BSP types (`portal_t`, `node_t`, `tree_t`, `plane_t`, `side_t`, etc.), map globals (`mapplanes`, `nummapplanes`, `entities`, `num_entities`), and winding utilities
- **`l_mem.h`**: `GetMemory`, `FreeMemory`, `MemorySize`
- **Defined elsewhere**: `BaseWindingForPlane`, `ChopWindingInPlace`, `ClipWindingEpsilon`, `FreeWinding`, `WindingIsTiny`, `WindingMemory`, `BaseWindingForNode` (winding ops from `l_poly`); `Log_Print`, `Log_Write` (logging); `Error`, `qprintf` (utility); `GetVectorForKey`, `ValueForKey` (entity key access); `numthreads`, `cancelconversion` (global compile-session flags); `DotProduct`, `VectorSubtract`, `VectorCopy`, `VectorCompare`, `ClearBounds`, `AddPointToBounds` (math macros/functions)

# code/bspc/prtfile.c
## File Purpose
Generates the `.prt` (portal file) used by the `qvis` visibility compiler. It traverses the BSP tree to enumerate vis-clusters and portals between them, writes the `PRT1`-format file, and stores cluster assignments back into the BSP leaf array.

## Core Responsibilities
- Recursively traverse the BSP tree to assign cluster numbers to leaf nodes
- Count vis-clusters and vis-portals for the portal file header
- Write portal geometry (winding points + cluster pair indices) to `name.prt`
- Handle detail-separator nodes as cluster boundaries (collapsing subtrees into one cluster)
- Rebuild portals suited for vis before writing (free old portals, create head-node portals, split)
- Propagate final cluster IDs back into `dleafs[]` after BSP write ordering

## External Dependencies
- **`qbsp.h`** — all core types (`node_t`, `portal_t`, `winding_t`, `tree_t`, `plane_t`) and declarations
- **`source`** (extern `char[1024]`, defined in `bspc.c`) — base filename for output path
- **`dleafs[]`** (defined in BSP file I/O layer, e.g. `l_bsp_q2.c`/`aas_file.c`) — output BSP leaf array
- **`Portal_VisFlood`**, **`MakeNodePortal`**, **`SplitNodePortals`**, **`MakeHeadnodePortals`**, **`Tree_FreePortals_r`** — defined in `portals.c` / `tree.c`
- **`WindingPlane`**, **`DotProduct`**, **`Q_rint`** — math utilities from `l_math.c` / `l_poly.c`
- **`Error`**, **`qprintf`** — logging/error utilities from `l_cmd.c`

# code/bspc/q2files.h
## File Purpose
Defines the on-disk binary file formats for Quake 2 assets, including PAK archives, PCX images, MD2 skeletal models, SP2 sprites, WAL textures, and the Q2 BSP map format. It is a read-only format specification header used by the BSPC tool to load and interpret legacy Quake 2 map data for AAS (Area Awareness System) generation.

## Core Responsibilities
- Define PAK archive header and file-entry structures for Q2 asset packages
- Declare the MD2 triangle-model binary layout (header, frames, verts, UVs, GL commands)
- Declare the SP2 sprite format structures
- Declare the WAL mip-texture format
- Define the Q2 BSP binary layout: lumps, planes, nodes, leaves, brushes, faces, edges, visibility
- Provide content flags (`CONTENTS_*`) and surface flags (`SURF_*`) for Q2 brush/surface classification
- Establish upper design bounds (`MAX_MAP_*`) for all BSP lump arrays

## External Dependencies
- No includes within this file; it depends on basic C types (`short`, `int`, `byte`, `float`, `char`) provided by the including translation unit's environment (typically via `qfiles.h` or a platform header).
- `byte` is assumed to be `unsigned char`, defined elsewhere (e.g., `q_shared.h`).
- `MAX_SKINNAME` (64) is defined locally and reused by both MD2 and SP2 structures.
- Content and surface flag constants (`CONTENTS_*`, `SURF_*`) mirror definitions in `q_shared.h` for the game module; `CONTENTS_Q2TRANSLUCENT` is explicitly renamed to avoid collision with the Q3 `CONTENTS_TRANSLUCENT` symbol.

# code/bspc/q3files.h
## File Purpose
Defines the binary file formats for Quake III Arena asset files used by the BSPC (BSP compiler) tool. It specifies on-disk data structures for MD3 triangle models and Q3 BSP map files, including all lump types, geometry limits, and layout constants.

## Core Responsibilities
- Define MD3 model format structures (frames, tags, surfaces, vertices, normals)
- Define Q3 BSP file format structures (header, lumps, nodes, leafs, brushes, surfaces)
- Enumerate all BSP lump indices and their count
- Provide capacity limits (`Q3_MAX_MAP_*`) for all BSP lump categories
- Provide capacity limits (`MD3_MAX_*`) for model geometry
- Define the draw surface type enum (`q3_mapSurfaceType_t`)

## External Dependencies
- No includes within this file; relies on `vec3_t` and `byte` being defined by an earlier include (typically `q_shared.h`) in any translation unit that includes this header.
- `vec3_t`, `byte`: defined elsewhere (q_shared.h / bg_public.h)

**Notes:**
- PCX and TGA struct definitions are commented out (block-comment syntax is broken — uses `* /` instead of `*/`), making them effectively dead documentation.
- MD3 normals use a compact spherical encoding packed into a single `short`.
- BSP plane pairs `(x&~1)` and `(x&~1)+1` are guaranteed opposites by convention (noted in comment).
- This file is nearly identical to `code/qcommon/qfiles.h` but with Q3-prefixed BSP names to avoid collisions in the BSPC tool, which also handles Quake 1/2/HL BSP formats.

# code/bspc/qbsp.h
## File Purpose
Central shared header for the BSPC (BSP Compiler) tool used to build AAS (Area Awareness System) navigation data from map BSP files. It defines all core BSP data structures, build-time constants, global state declarations, and the full inter-module function API surface used across the BSPC pipeline.

## Core Responsibilities
- Defines fundamental BSP construction types (`plane_t`, `side_t`, `mapbrush_t`, `face_t`, `bspbrush_t`, `node_t`, `portal_t`, `tree_t`)
- Declares global build-control flags (`noprune`, `nodetail`, `nomerge`, `create_aas`, etc.)
- Declares global map storage arrays and counters (`mapplanes`, `mapbrushes`, `brushsides`, etc.)
- Enumerates supported map source formats (Q1, Q2, Q3, Half-Life, Sin)
- Provides the inter-module function declaration surface for: map loading, CSG, BSP construction, portalization, tree management, leak detection, GL debug output, and texture resolution
- Aggregates all BSPC-local headers into a single include point so translation units need only `#include "qbsp.h"`

## External Dependencies

- `l_cmd.h` — command-line utilities, `qboolean`, file I/O helpers
- `l_math.h` — `vec3_t`, `vec_t`, vector math macros/functions
- `l_poly.h` — `winding_t`, convex polygon operations
- `l_threads.h` — threading and mutex primitives
- `../botlib/l_script.h` — lexical script parser (used by map loaders)
- `l_bsp_ent.h` — BSP entity (`entity_t`) type (defined elsewhere)
- `q2files.h` — Quake 2 BSP on-disk format definitions
- `l_mem.h`, `l_utils.h`, `l_log.h`, `l_qfiles.h` — memory, utility, logging, pak-file helpers
- `<io.h>` (Win32), `<malloc.h>` — platform allocation
- `quakefile_s` — forward-declared struct used by `LoadMapFromBSP` / `Q3_LoadMapFromBSP` (defined in `l_qfiles.h`)
- `tmp_face_s` — forward-declared in `portal_t`; defined in AAS/BSPC build code elsewhere

# code/bspc/qfiles.h
## File Purpose
Defines binary on-disk file formats for Quake 2-era assets used by the BSPC (BSP Compiler) tool. It covers PAK archives, PCX images, MD2 models, SP2 sprites, WAL textures, and the Q2 BSP map format. This is a legacy format header distinct from the Q3 BSP structures used at runtime.

## Core Responsibilities
- Define magic number constants (FourCC identifiers) for each file format
- Declare packed structs that map directly to binary file layouts
- Enumerate BSP lump indices and upper-bound limits for map data arrays
- Define surface flags (`SURF_*`) and content flags (`CONTENTS_*`) for brush/leaf classification
- Provide the complete Q2 BSP in-memory/on-disk structural hierarchy (header → lumps → geometry)

## External Dependencies
- No `#include` directives in this header; relies on the including translation unit to provide `byte`, `short`, `int` primitive typedefs (typically from `qfiles.h` or `q_shared.h` up the include chain).
- `MAX_SKINNAME` (64) is defined within this file and reused by both `dmdl_t` skin name storage and `dsprframe_t`.
- All `CONTENTS_*` and `SURF_*` flags are noted as needing to stay in sync with `q_shared.h` for the runtime engine.

# code/bspc/sinfiles.h
## File Purpose
Defines the binary BSP file format structures and constants for the SIN engine (a Quake II-derived game). It is used by the BSPC tool to read and process SIN-format `.bsp` files for bot navigation area (AAS) generation.

## Core Responsibilities
- Define the SIN BSP version constant (`SINBSPVERSION 41`) and all map size upper bounds
- Declare lump index constants and the file header structure for SIN BSP files
- Provide geometry structures: planes, vertices, nodes, faces, leaves, edges, brushes
- Define SIN-specific surface flags (rendering, physics, material type) and content flags
- Declare the `sin_texinfo_t` structure with extended SIN-specific texture/surface properties
- Define visibility (`sin_dvis_t`) and area portal structures for PVS/PHS data
- Encode surface material types (wood, metal, stone, etc.) in the upper 4 bits of surface flags via `SURFACETYPE_FROM_FLAGS`

## External Dependencies
- `vec3_t` — defined in a shared math/types header (e.g., `mathlib.h` or `q_shared.h`); not defined here
- `byte` — platform typedef, defined elsewhere
- `MAXLIGHTMAPS` — redefined here to 16 (undefed first to override any prior definition)
- Conditional compilation entirely controlled by `#define SIN` (set at the top of this file itself)

# code/bspc/tetrahedron.c
## File Purpose
Implements a tetrahedral decomposition algorithm for the BSPC tool, converting an AAS (Area Awareness System) world's solid faces into a triangle mesh and then subdividing that mesh into tetrahedrons. This is a spatial decomposition utility used during AAS file processing.

## Core Responsibilities
- Allocate and free the global `thworld` data store for tetrahedron construction
- Manage hashed pools of vertices, planes, edges, triangles, and tetrahedrons with find-or-create semantics
- Validate candidate edges and triangles against existing geometry to prevent intersections
- Search for valid tetrahedrons using two strategies: shared-edge pairing (`TH_FindTetrahedron1`) and single-triangle + free vertex (`TH_FindTetrahedron2`)
- Drive the full decomposition loop (`TH_TetrahedralDecomposition`) until no new tetrahedrons can be formed
- Convert AAS solid faces into triangles (`TH_CreateAASFaceTriangles`, `TH_AASToTriangleMesh`)
- Provide the top-level entry point `TH_AASToTetrahedrons` that loads an AAS file and runs the full pipeline

## External Dependencies
- `qbsp.h` — BSP types, math macros (`DotProduct`, `CrossProduct`, `VectorNormalize`, etc.), `Error`, `qprintf`, `Log_Print`
- `l_mem.h` — `GetClearedMemory`, `FreeMemory`
- `botlib/aasfile.h` — `aas_face_t`, `aas_edge_t`, `FACE_SOLID`, AAS data structure definitions
- `aas_store.h` — `aasworld` global (type `aas_t`), AAS area/face/edge/vertex arrays
- `aas_cfg.h` — configuration types (included transitively, not directly used here)
- `aas_file.h` — `AAS_LoadAASFile` (defined elsewhere)
- `aasworld` — global AAS world state (defined in `be_aas_def` / `aas_store`)

# code/bspc/tetrahedron.h
## File Purpose
Header file declaring a single utility function for converting AAS (Area Awareness System) data into a tetrahedral representation. It serves as the public interface for `tetrahedron.c` within the BSPC (BSP Compiler) tool.

## Core Responsibilities
- Exposes one conversion function to other BSPC translation units
- Acts as the sole interface between AAS data and tetrahedral geometry output

## External Dependencies
- No includes in this header.
- `TH_AASToTetrahedrons` is defined in `code/bspc/tetrahedron.c` (defined elsewhere).

# code/bspc/textures.c
## File Purpose
Provides texture resolution and texinfo generation utilities for the BSPC (BSP Compiler) tool. It maps brush surface texture names to miptex metadata and computes axis-aligned texture projection vectors used when writing BSP lumps.

## Core Responsibilities
- Cache and deduplicate loaded miptex entries (name, flags, value, contents, animation chain)
- Load `.wal` texture files from disk to extract surface flags/contents metadata
- Compute texture projection axes from a plane normal using a best-fit axis table
- Build and deduplicate `texinfo_t` records from brush texture parameters (scale, rotate, shift, origin offset)
- Recursively resolve animated texture chains via `nexttexinfo` linkage

## External Dependencies
- `qbsp.h` — `plane_t`, `brush_texture_t`, `textureref_t`, `MAX_MAP_TEXTURES`, common BSPC types and declarations
- `l_bsp_q2.h` — `texinfo_t`, `numtexinfo`, global BSP lump arrays (`texinfo[]`)
- `q2files.h` (via `qbsp.h`) — `miptex_t` on-disk layout, `texinfo_t` definition
- `TryLoadFile`, `FreeMemory` — defined in memory/file utility modules (`l_mem`, `l_qfiles`)
- `Error` — fatal error handler, defined elsewhere
- `gamedir` — global string for game data path, defined elsewhere
- `DotProduct`, `VectorCopy` — math macros/functions from `l_math.h`
- `LittleLong` — endian swap macro, defined elsewhere

# code/bspc/tree.c
## File Purpose
Manages the lifecycle and traversal of BSP trees used in the BSPC (BSP Compiler) tool. Provides allocation, deallocation, traversal, debug printing, and pruning of BSP tree nodes and their associated portals and brushes.

## Core Responsibilities
- Allocate and zero-initialize `tree_t` structures
- Recursively free portals attached to BSP nodes
- Recursively free brush lists, volume brushes, and node memory
- Traverse the BSP tree to find the leaf node containing a given 3D point
- Debug-print the BSP tree structure to stdout
- Prune redundant interior nodes where both children are solid (optimization pass)

## External Dependencies
- **`qbsp.h`** — defines all core BSP types (`tree_t`, `node_t`, `portal_t`, `bspbrush_t`, `plane_t`), constants (`PLANENUM_LEAF`, `CONTENTS_SOLID`, `CONTENTS_LADDER`), and global arrays (`mapplanes`).
- **Defined elsewhere:** `RemovePortalFromNode` (portals.c), `FreePortal` (portals.c), `FreeBrush`/`FreeBrushList` (brushbsp.c), `GetMemory`/`FreeMemory`/`MemorySize` (l_mem.c), `ClearBounds` (l_math/l_cmd), `Log_Print`/`PrintMemorySize` (l_log.c), `numthreads` (l_threads.c), `create_aas` (bspc.c).

# code/bspc/writebsp.c
## File Purpose
Serializes the in-memory BSP tree and associated map data (planes, faces, leaves, nodes, brushes, models) into the flat arrays used by the BSP file format. It is the final output stage of the BSPC compiler before the binary file is written to disk.

## Core Responsibilities
- Walk the BSP node tree recursively and emit nodes, leaves, and faces into output arrays
- Emit planes, brushes, and brush sides (including axis-aligned bevel planes for collision)
- Assign leaf-face and leaf-brush index ranges
- Bookend model compilation with `BeginModel`/`EndModel` to track face/leaf ranges per entity model
- Assign `model` key-value pairs to brush entities (`SetModelNumbers`)
- Assign unique light style numbers to targetname-controlled lights (`SetLightStyles`)
- Initialize and finalize global BSP file counters (`BeginBSPFile`/`EndBSPFile`)

## External Dependencies
- **Includes:** `qbsp.h` (pulls in all BSPC-internal headers, q2files.h for BSP output types, l_bsp_ent.h for entity helpers)
- **Defined elsewhere:**
  - `dplanes[]`, `dfaces[]`, `dleafs[]`, `dnodes[]`, `dbrushes[]`, `dbrushsides[]`, `dleaffaces[]`, `dleafbrushes[]`, `dsurfedges[]`, `dmodels[]`, `numplanes`, `numfaces`, `numleafs`, `numnodes`, `numbrushes`, `numbrushsides`, `numleaffaces`, `numleafbrushes`, `numsurfedges`, `numedges`, `numvertexes`, `nummodels` — global BSP output arrays/counters (q2files.h / bspc global state)
  - `mapplanes`, `nummapplanes`, `mapbrushes`, `nummapbrushes`, `entities`, `num_entities`, `entity_num` — map loading globals
  - `GetEdge2` — edge deduplication (faces.c or similar)
  - `EmitAreaPortals` — portals.c
  - `Q2_UnparseEntities` — map_q2.c
  - `FindFloatPlane` — map.c
  - `ValueForKey`, `SetKeyValue` — l_bsp_ent.c
  - `qprintf`, `Error` — l_cmd.c

# code/cgame/cg_consolecmds.c
## File Purpose
Registers and dispatches client-side console commands typed at the local console or bound to keys. It bridges player input (keyboard bindings, console text) to cgame actions such as score display, weapon cycling, team orders, and voice chat.

## Core Responsibilities
- Defines a static dispatch table (`commands[]`) mapping command name strings to handler functions
- Implements `CG_ConsoleCommand` to look up and invoke handlers when the engine forwards an unrecognized command to cgame
- Implements `CG_InitConsoleCommands` to register all commands with the engine for tab-completion
- Handles scoreboard show/hide state and optional score refresh requests
- Provides tell/voice-tell shortcuts targeting crosshair player or last attacker
- Under `MISSIONPACK`: handles HUD reloading, team orders, scoreboard scrolling, and SP win/lose sequences

## External Dependencies
- `cg_local.h` — full cgame state (`cg_t cg`, `cgs_t cgs`), trap declarations, and function prototypes
- `../ui/ui_shared.h` — `menuDef_t`, `Menu_ScrollFeeder`, `String_Init`, `Menu_Reset`
- **Defined elsewhere:** `CG_CrosshairPlayer`, `CG_LastAttacker`, `CG_LoadMenus`, `CG_AddBufferedSound`, `CG_CenterPrint`, `CG_BuildSpectatorString`, `CG_SelectNextPlayer`/`CG_SelectPrevPlayer`, `CG_OtherTeamHasFlag`, `CG_YourTeamHasFlag`, `CG_LoadDeferredPlayers`, all `CG_TestModel_*`/`CG_Zoom*`/`CG_*Weapon_f` functions, all `trap_*` syscalls

# code/cgame/cg_draw.c
## File Purpose
Implements all 2D and some 3D HUD rendering for the cgame module during active gameplay. It draws the status bar, crosshair, lagometer, scoreboards, team overlays, center prints, and all other screen-space UI elements composited over the 3D world view.

## Core Responsibilities
- Render the player status bar (health, armor, ammo, weapon model icons)
- Draw the crosshair and crosshair entity name labels
- Display team overlay, scores, powerup timers, and pickup notifications
- Render the lagometer (frame interpolation + snapshot latency graph)
- Handle center-print messages with fade timing
- Orchestrate the full 2D draw pass (`CG_Draw2D`) called each frame after 3D scene render
- Drive the top-level `CG_DrawActive` entry point for stereo-aware full-screen rendering

## External Dependencies
- `cg_local.h` — all cgame types, `cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items`, trap declarations
- `ui/ui_shared.h` (MISSIONPACK) — `displayContextDef_t`, `menuDef_t`, `Menu_Paint`, `Menus_FindByName`
- **Defined elsewhere:** `CG_DrawOldScoreboard`, `CG_DrawOldTourneyScoreboard`, `CG_DrawWeaponSelect`, `CG_DrawStringExt`, `CG_DrawBigString`, `CG_FadeColor`, `CG_ColorForHealth`, `CG_AdjustFrom640`, `BG_FindItemForPowerup`, `trap_R_*`, `trap_S_*`, `trap_CM_*`, `g_color_table`, `colorWhite`, `colorBlack`

# code/cgame/cg_drawtools.c
## File Purpose
Provides low-level 2D rendering helper functions for the cgame module, including coordinate scaling, filled/outlined rectangles, image blitting, character/string rendering, and HUD utility queries. All functions operate in a virtual 640×480 coordinate space and scale to the actual display resolution.

## Core Responsibilities
- Scale 640×480 virtual coordinates to real screen pixels via `cgs.screenXScale`/`screenYScale`
- Draw filled rectangles, bordered rectangles, and textured quads
- Render individual characters and multi-style strings (color codes, shadows, proportional fonts, banner fonts)
- Tile background graphics around a reduced viewport
- Compute time-based fade alpha and team color vectors
- Map health/armor values to a color gradient for HUD display

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere**: `cg` (`cg_t`), `cgs` (`cgs_t`), `g_color_table`, `Q_IsColorString`, `ColorIndex`, `VectorClear`
- **Trap calls**: `trap_R_SetColor`, `trap_R_DrawStretchPic` (renderer syscalls)
- **Constants used**: `FADE_TIME`, `PULSE_DIVISOR`, `PROP_*`, `PROPB_*`, `UI_CENTER/RIGHT/DROPSHADOW/INVERSE/PULSE/SMALLFONT/FORMATMASK`, `ARMOR_PROTECTION`, `BIGCHAR_*`, `SMALLCHAR_*`

# code/cgame/cg_effects.c
## File Purpose
Generates client-side visual effects as local entities, primarily in response to game events such as weapon impacts, player deaths, teleportation, and special item activations. All effects are purely cosmetic and client-local, not networked.

## Core Responsibilities
- Spawn bubble trail local entities for underwater projectiles
- Create smoke puff / blood trail local entities with configurable color, fade, and velocity
- Generate explosion local entities (sprite and model variants)
- Spawn player gib fragments with randomized gravity trajectories
- Handle teleport, score plum, and MissionPack-exclusive effects (Kamikaze, Obelisk, Invulnerability)
- Emit positional sounds for pain and impact events (Obelisk, Invulnerability)

## External Dependencies
- `cg_local.h`: All cgame types (`localEntity_t`, `cg_t`, `cgs_t`, `leType_t`, etc.)
- `cg.time`: Current client render time (global `cg_t`)
- `cgs.media.*`: Preloaded shader/model handles (global `cgs_t`)
- `cgs.glconfig.hardwareType`: GPU capability check for RagePro fallback
- **Defined elsewhere**: `CG_AllocLocalEntity` (`cg_localents.c`), `CG_MakeExplosion` (this file, called by `CG_ObeliskExplode`), `trap_S_StartSound` (syscall layer), `AxisClear`, `RotateAroundDirection`, `VectorNormalize`, `AnglesToAxis` (math library), `axisDefault` (global defined in renderer/shared code)

# code/cgame/cg_ents.c
## File Purpose
Presents server-transmitted snapshot entities to the renderer and sound system every frame. It resolves interpolated/extrapolated positions for all `centity_t` objects and dispatches per-type rendering logic (players, missiles, movers, items, etc.).

## Core Responsibilities
- Compute per-frame lerp/extrapolated origins and angles for all packet entities via `CG_CalcEntityLerpPositions`
- Apply continuous per-entity effects (looping sounds, constant lights) via `CG_EntityEffects`
- Dispatch entity-type-specific rendering through `CG_AddCEntity` (switch on `eType`)
- Attach child render entities to parent model tags (`CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag`)
- Adjust entity positions when riding movers (`CG_AdjustPositionForMover`)
- Drive the auto-rotation state (`cg.autoAngles/autoAxis`) used by all world items
- Submit the local predicted player entity in addition to server-sent entities

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `tr_types.h`, `cg_public.h`
- **Defined elsewhere:** `CG_Player` (`cg_players.c`), `CG_AddRefEntityWithPowerups` (`cg_players.c`), `CG_GrappleTrail` (`cg_weapons.c`), `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta`, `BG_PlayerStateToEntityState` (`bg_misc.c`/`bg_pmove.c`)
- Renderer traps: `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_LerpTag`
- Sound traps: `trap_S_UpdateEntityPosition`, `trap_S_AddLoopingSound`, `trap_S_AddRealLoopingSound`, `trap_S_StartSound`
- Math utilities: `VectorCopy/MA/Add/Subtract/Scale/Clear/Normalize2`, `AnglesToAxis`, `MatrixMultiply`, `AxisCopy/Clear`, `RotateAroundDirection`, `PerpendicularVector`, `CrossProduct`, `ByteToDir`, `LerpAngle`

# code/cgame/cg_event.c
## File Purpose
Handles client-side entity event processing at snapshot transitions and playerstate changes. It translates server-generated event codes into audio, visual, and HUD feedback for the local client. This is the primary event dispatch hub for the cgame module.

## Core Responsibilities
- Dispatch `EV_*` events from entity states to appropriate audio/visual handlers
- Display kill obituary messages in the console and center-print frags to the killer
- Handle item pickup notification, weapon selection, and holdable item usage
- Manage movement feedback: footsteps, fall sounds, step smoothing, jump pad effects
- Route CTF/team-mode global sound events with team-context-aware sound selection
- Forward missile impact, bullet, railgun, and shotgun events to weapon effect functions
- Gate event re-firing by tracking `previousEvent` on each `centity_t`

## External Dependencies
- `cg_local.h` — all cgame types, globals (`cg`, `cgs`), and trap declarations.
- `ui/menudef.h` — `VOICECHAT_*` constants (MissionPack only).
- **Defined elsewhere:** `BG_EvaluateTrajectory`, `BG_FindItemForHoldable`, `ByteToDir`, `Info_ValueForKey`, `Q_strncpyz`, `Com_sprintf`, `va`; all `CG_*` effect/weapon functions; all `trap_S_*` sound traps.

# code/cgame/cg_info.c
## File Purpose
Implements the loading screen (info screen) displayed while a Quake III Arena level is being loaded. It renders a level screenshot background, player/item icons accumulated during asset loading, and various server/game metadata strings.

## Core Responsibilities
- Accumulate player and item icon handles as clients and items are registered during map load
- Display a loading progress string updated in real time via `trap_UpdateScreen`
- Render the level screenshot backdrop with a detail texture overlay
- Draw server metadata: hostname, pure-server status, MOTD, map message, cheat warning
- Display game type and rule limits (timelimit, fraglimit, capturelimit)
- Register player model icons and, in single-player, pre-cache personality announce sounds

## External Dependencies
- **Includes**: `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere**: `bg_itemlist` (game shared item table), `CG_ConfigString`, `CG_DrawPic`, `UI_DrawProportionalString`, `trap_R_RegisterShaderNoMip`, `trap_R_RegisterShader`, `trap_R_DrawStretchPic`, `trap_R_SetColor`, `trap_S_RegisterSound`, `trap_UpdateScreen`, `trap_Cvar_VariableStringBuffer`, `Q_strncpyz`, `Q_strrchr`, `Q_CleanStr`, `Info_ValueForKey`, `Com_sprintf`, `va`, `atoi`

# code/cgame/cg_local.h
## File Purpose
Central private header for the Quake III Arena cgame (client-game) module. Defines all major data structures, global state declarations, cvar externs, and function prototypes used across every cgame source file. Acts as the single shared contract binding all cgame subsystems together.

## Core Responsibilities
- Define timing/animation/display constants for client-side visual effects
- Declare `centity_t`, `cg_t`, `cgs_t`, `cgMedia_t`, `clientInfo_t`, `weaponInfo_t`, and related types
- Declare all cgame-module-global extern variables (`cg`, `cgs`, `cg_entities`, `cg_weapons`, etc.)
- Expose all vmCvar externs used by cgame subsystems
- Prototype all public functions across cgame `.c` files (draw, predict, players, weapons, effects, marks, etc.)
- Declare all engine system trap functions (`trap_*`) that bridge the VM to the main executable

## External Dependencies
- `../game/q_shared.h` — shared math, string, entity/player state, cvar, trace types
- `tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t`, etc.)
- `../game/bg_public.h` — gameplay constants, animation enums, `gitem_t`, `pmove_t`, `playerState_t`
- `cg_public.h` — `snapshot_t`, `cgameImport_t`/`cgameExport_t` enums, import API version
- All `trap_*` symbols: defined in engine executable, resolved at VM load time (not in this file)
- `gitem_t bg_itemlist[]`, `Pmove()`: defined in `bg_*.c` shared game code

# code/cgame/cg_localents.c
## File Purpose
Manages a fixed-size pool of client-side "local entities" (smoke puffs, gibs, brass shells, explosions, score plums, etc.) that exist purely on the client and are never synchronized with the server. Every frame, it iterates all active local entities and submits renderer commands appropriate to each entity type.

## Core Responsibilities
- Maintain a pool of 512 `localEntity_t` slots via a doubly-linked active list and a singly-linked free list
- Allocate and free local entities, evicting the oldest active entity when the pool is exhausted
- Simulate fragment physics: trajectory evaluation, collision tracing, bounce/reflect, mark/sound generation, and ground-sinking
- Drive per-type visual update functions (fade, scale, fall, explosion, sprite explosion, score plum, kamikaze, etc.)
- Submit all live local entities to the renderer each frame via `trap_R_AddRefEntityToScene`

## External Dependencies
- **Includes:** `cg_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:**
  - `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — `bg_pmove.c`
  - `CG_SmokePuff`, `CG_ImpactMark` — `cg_effects.c`, `cg_marks.c`
  - `CG_Trace` — `cg_predict.c`
  - `CG_GibPlayer` — `cg_effects.c`
  - `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_CM_PointContents`, `trap_S_StartSound`, `trap_S_StartLocalSound` — engine syscall layer
  - `cg`, `cgs` — global state in `cg_main.c`

# code/cgame/cg_main.c
## File Purpose
This is the primary entry point and initialization module for the cgame (client-side game) VM module in Quake III Arena. It owns all global cgame state, registers cvars, and orchestrates the full asset precache pipeline during level load.

## Core Responsibilities
- Expose `vmMain()` as the sole entry point from the engine into the cgame VM
- Declare and own all global cgame state (`cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items`)
- Register and update all cgame `vmCvar_t` variables via a data-driven table
- Drive the level initialization sequence: sounds → graphics → clients → HUD
- Provide utility functions: `CG_Printf`, `CG_Error`, `CG_Argv`, `CG_ConfigString`
- Implement stub `Com_Error`/`Com_Printf` linkage for shared `q_shared.c`/`bg_*.c` code
- (MISSIONPACK) Load and initialize the script-driven HUD menu system via `displayContextDef_t`

## External Dependencies
- `cg_local.h` — pulls in `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`, and all `trap_*` declarations
- `ui/ui_shared.h` (MISSIONPACK) — `displayContextDef_t`, menu system types, `Init_Display`, `Menu_*`, `PC_*` parse helpers
- **Defined elsewhere:** `CG_DrawActiveFrame`, `CG_ConsoleCommand`, `CG_NewClientInfo`, `CG_RegisterItemVisuals`, `CG_ParseServerinfo`, `CG_SetConfigValues`, `bg_itemlist`, `bg_numItems`, all `trap_*` syscall stubs

# code/cgame/cg_marks.c
## File Purpose
Manages persistent wall mark decals (bullet holes, burn marks, blood splats) and a full particle simulation system for the cgame module. Despite the filename, the file contains two logically separate systems: mark polys and a Ridah-era particle engine that was folded in.

## Core Responsibilities
- Maintain a fixed-size pool of `markPoly_t` nodes using a doubly-linked active list and singly-linked free list
- Project impact decals onto world geometry by clipping a quad against BSP surfaces via `trap_CM_MarkFragments`
- Fade and expire persistent mark polys each frame, submitting survivors to the renderer
- Maintain a fixed-size pool of `cparticle_t` particles (weather, smoke, blood, bubbles, sprites, animations)
- Update and submit particles each frame with physics integration (velocity + acceleration)
- Provide factory functions for spawning typed particles (snow, smoke, sparks, blood, explosions, etc.)

## External Dependencies
- `cg_local.h` — all cgame types, globals (`cg`, `cgs`), and trap declarations
- `trap_CM_MarkFragments` — BSP polygon clipping (defined in engine)
- `trap_R_AddPolyToScene` — renderer polygon submission (defined in engine)
- `trap_R_RegisterShader` — shader registration for particle anim frames (defined in engine)
- `VectorNormalize2`, `PerpendicularVector`, `RotatePointAroundVector`, `CrossProduct`, `VectorMA`, `DotProduct`, `Distance`, `AngleVectors`, `vectoangles` — math utilities (defined in `q_math.c`/`q_shared.c`)
- `cg_addMarks` cvar — gates mark/particle submission; declared in `cg_main.c`
- `cgs.media.energyMarkShader`, `tracerShader`, `smokePuffShader`, `waterBubbleShader` — preloaded media handles

# code/cgame/cg_newdraw.c
## File Purpose
MissionPack (Team Arena)-exclusive HUD drawing module for the cgame client. It implements all "owner draw" HUD element renderers for team game UI elements (health, armor, flags, team overlay, spectator ticker, medals, etc.) and handles mouse/keyboard input routing to the UI display system.

## Core Responsibilities
- Render individual HUD elements via a central `CG_OwnerDraw` dispatch function keyed on owner-draw enum constants
- Display team-specific overlays: selected player health/armor/status/weapon/head, flag status, team scores
- Manage team-ordered player selection (`CG_SelectNextPlayer`, `CG_SelectPrevPlayer`) and pending order dispatch
- Animate the local player's head portrait with damage reaction and idle bobbing
- Draw the scrolling spectator ticker and team chat/system chat areas
- Draw end-of-round medal statistics (accuracy, assists, gauntlet, captures, etc.)
- Route mouse movement and key events to the shared UI `Display_*` system

## External Dependencies
- **Includes:** `cg_local.h`, `../ui/ui_shared.h`
- **External symbols used but defined elsewhere:**
  - `cgDC` (`displayContextDef_t`) — defined in `cg_main.c`
  - `sortedTeamPlayers[]`, `numSortedTeamPlayers` — defined in `cg_draw.c`
  - `systemChat`, `teamChat1`, `teamChat2` — defined in `cg_draw.c`
  - `cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items` — cgame globals
  - `BG_FindItemForPowerup` — game/bg_misc.c
  - `Display_*`, `Menus_*` — ui/ui_shared.c
  - All `trap_*` functions — cgame VM syscall stubs

# code/cgame/cg_particles.c
## File Purpose
Implements a software particle system for the cgame module, managing a fixed pool of particles that simulate weather (snow, flurry, bubbles), combat effects (blood, smoke, sparks, explosions), and environmental effects (oil slicks, dust). Particles are submitted each frame as raw polygons to the renderer via `trap_R_AddPolyToScene`.

## Core Responsibilities
- Maintain a free-list / active-list pool of `MAX_PARTICLES` (8192) particles
- Initialize and register animated shader sequences used by explosion/anim particles
- Classify particles by type and build camera-aligned or flat polygon geometry each frame
- Apply simple physics (position = origin + vel*t + accel*t²) during the update pass
- Cull expired particles back to the free list; cull distant particles to avoid poly budget overruns
- Provide typed spawn helpers called from other cgame subsystems (weapons, events, entities)

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `cg` (global `cg_t`), `cgs` (global `cgs_t`) — read for time, refdef, snapshot, media handles, GL config
- `trap_R_RegisterShader`, `trap_R_AddPolyToScene` — renderer syscalls (defined in cgame syscall layer)
- `trap_CM_BoxTrace` (via `CG_Trace`) — used in `ValidBloodPool`
- `crandom`, `random`, `VectorMA`, `VectorCopy`, `vectoangles`, `AngleVectors`, `Distance`, `VectorLength`, `VectorNegate`, `VectorClear`, `VectorSet`, `DEG2RAD` — defined in `q_shared`/`q_math`
- `COM_Parse`, `stricmp`, `atoi`, `atof`, `va`, `memset` — standard/engine string utilities
- `CG_ConfigString`, `CG_Printf`, `CG_Error`, `CG_Trace` — defined elsewhere in cgame

# code/cgame/cg_players.c
## File Purpose
Handles all client-side player entity rendering for Quake III Arena, including model/skin/animation loading, deferred client info management, per-frame skeletal animation evaluation, and visual effect attachment (powerups, flags, shadows, sprites).

## Core Responsibilities
- Load and cache per-client 3-part player models (legs/torso/head) with skins and animations
- Parse `animation.cfg` files to populate animation tables for each player model
- Manage deferred loading of client info to avoid hitches during gameplay
- Evaluate and interpolate animation lerp frames for legs, torso, and flag each frame
- Compute per-frame player orientation (yaw swing, pitch, roll lean, pain twitch)
- Assemble and submit the full player refEntity hierarchy to the renderer
- Attach visual effects: powerup overlays, flag models, haste/breath/dust trails, shadow marks, floating sprites

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `COM_Parse`, `Info_ValueForKey`, `Q_stricmp`, `Q_strncpyz`, `VectorCopy/Clear/Set/MA/Normalize`, `DotProduct`, `AnglesToAxis`, `AngleMod`, `BG_EvaluateTrajectory` — defined in shared/game code
- `trap_*` functions — VM syscall stubs defined in `cg_syscalls.c`
- `CG_SmokePuff`, `CG_ImpactMark`, `CG_PositionRotatedEntityOnTag`, `CG_PositionEntityOnTag`, `CG_AddPlayerWeapon` — defined in other cgame modules
- `cgs`, `cg`, `cg_entities` — global state defined in `cg_main.c`

# code/cgame/cg_playerstate.c
## File Purpose
Processes transitions between consecutive `playerState_t` snapshots on the client side, driving audio feedback, visual damage effects, event dispatch, and UI state updates whenever the local player's state changes. Works for both live prediction and demo/follow-cam playback.

## Core Responsibilities
- Compute and set low-ammo warning level (`CG_CheckAmmo`)
- Calculate screen-shake direction and magnitude from incoming damage (`CG_DamageFeedback`)
- Handle respawn bookkeeping (`CG_Respawn`)
- Dispatch playerstate-embedded events into the entity event system (`CG_CheckPlayerstateEvents`)
- Detect and re-fire predicted events that were corrected by the server (`CG_CheckChangedPredictableEvents`)
- Play context-sensitive local/announcer sounds for hits, kills, rewards, timelimit, and fraglimit (`CG_CheckLocalSounds`)
- Orchestrate all of the above on each snapshot transition (`CG_TransitionPlayerState`)

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere:** `CG_EntityEvent`, `CG_PainEvent`, `CG_AddBufferedSound`, `AngleVectors`, `DotProduct`, `VectorSubtract`, `VectorLength`, `trap_S_StartLocalSound`, `cg`, `cgs`, `cg_entities`, `cg_showmiss`

# code/cgame/cg_predict.c
## File Purpose
Generates `cg.predictedPlayerState` each frame by either interpolating between two server snapshots or running local client-side `Pmove` prediction on unacknowledged user commands. Also provides collision query utilities used by the prediction physics.

## Core Responsibilities
- Build a filtered sublist of solid and trigger entities from the current snapshot for efficient collision tests
- Provide `CG_Trace` and `CG_PointContents` wrappers that test against both world BSP and solid entities
- Interpolate player state between two snapshots when prediction is disabled or in demo playback
- Run client-side `Pmove` on all unacknowledged commands to predict the local player's position ahead of server acknowledgement
- Detect and decay prediction errors caused by server-vs-client divergence
- Predict item pickups and trigger interactions (jump pads, teleporters) locally

## External Dependencies
- **Includes:** `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:**
  - `Pmove` — `bg_pmove.c`
  - `BG_EvaluateTrajectory`, `BG_PlayerTouchesItem`, `BG_CanItemBeGrabbed`, `BG_TouchJumpPad`, `BG_AddPredictableEventToPlayerstate`, `PM_UpdateViewAngles` — `bg_*.c`
  - `CG_AdjustPositionForMover`, `CG_TransitionPlayerState` — `cg_ents.c`, `cg_playerstate.c`
  - All `trap_CM_*` functions — cgame syscall layer (`cg_syscalls.c`)
  - `cg`, `cgs`, `cg_entities[]` — `cg_main.c`

# code/cgame/cg_public.h
## File Purpose
Defines the public interface contract between the cgame module (client-side game logic) and the main engine executable. It declares the snapshot data structure and enumerates all syscall IDs for both engine-to-cgame (imported) and cgame-to-engine (exported) function dispatch tables.

## Core Responsibilities
- Define `snapshot_t`, the primary unit of server-state delivery to the client
- Enumerate all engine services available to the cgame VM via `cgameImport_t` trap IDs
- Enumerate all cgame entry points callable by the engine via `cgameExport_t`
- Define `CMD_BACKUP` / `CMD_MASK` constants for the client command ring buffer
- Declare `CGAME_IMPORT_API_VERSION` for ABI compatibility checking
- Declare cgame UI event type constants (`CGAME_EVENT_*`)

## External Dependencies
- `MAX_MAP_AREA_BYTES` — defined in `qcommon/qfiles.h` or `game/q_shared.h`
- `playerState_t` — defined in `game/bg_public.h`
- `entityState_t` — defined in `game/q_shared.h` / `game/bg_public.h`
- `byte`, `qboolean`, `stereoFrame_t` — defined in `game/q_shared.h`
- `SNAPFLAG_*` constants — defined elsewhere (likely `qcommon/qcommon.h`)
- All `cgameImport_t` trap implementations — defined in `client/cl_cgame.c` (`CL_CgameSystemCalls`)
- All `cgameExport_t` entry points — implemented in `cgame/cg_main.c` (`vmMain`)

# code/cgame/cg_scoreboard.c
## File Purpose
Renders the in-game scoreboard overlay for Quake III Arena, including both the standard mid-game scoreboard and the oversized tournament intermission scoreboard. It handles FFA, team, and spectator layouts with fade animations.

## Core Responsibilities
- Draw per-client score rows with bot icons, player heads, flag indicators, and score/ping/time/name text
- Handle adaptive layout switching between normal and interleaved (compact) modes based on player count
- Render ranked team scoreboards in correct lead order (leading team drawn first)
- Display killer name, current rank/score string, and team score comparison at top of screen
- Draw scoreboard column headers (score/ping/time/name icons)
- Render the full-screen tournament scoreboard with giant text for MOTD, server time, and player scores
- Ensure the local client is always visible, appending their row at the bottom if scrolled off

## External Dependencies
- `cg_local.h` — all shared cgame types, globals (`cg`, `cgs`), and function declarations
- **Defined elsewhere:** `CG_DrawFlagModel`, `CG_DrawPic`, `CG_DrawHead`, `CG_FillRect`, `CG_DrawBigString`, `CG_DrawBigStringColor`, `CG_DrawSmallStringColor`, `CG_DrawStringExt`, `CG_DrawStrlen`, `CG_FadeColor`, `CG_PlaceString`, `CG_DrawTeamBackground`, `CG_LoadDeferredPlayers`, `CG_ConfigString`, `trap_SendClientCommand`, `Com_Printf`, `Com_sprintf`
- Constants `SB_NORMAL_HEIGHT`, `SB_INTER_HEIGHT`, `SB_MAXCLIENTS_NORMAL`, `SB_MAXCLIENTS_INTER`, `SB_SCORELINE_X`, etc. are all `#define`d locally in this file.

# code/cgame/cg_servercmds.c
## File Purpose
Handles reliably-sequenced text commands sent by the server to the cgame module. All commands are processed at snapshot transition time, guaranteeing a valid snapshot is present. Also manages the voice chat system including parsing, buffering, and playback.

## Core Responsibilities
- Dispatch incoming server commands (`cp`, `cs`, `print`, `chat`, `tchat`, `scores`, `tinfo`, `map_restart`, etc.) via `CG_ServerCommand`
- Parse and apply score data (`CG_ParseScores`) and team overlay info (`CG_ParseTeamInfo`)
- Parse and cache server configuration strings (`CG_ParseServerinfo`, `CG_SetConfigValues`)
- Handle config-string change notifications and re-register models/sounds/client info accordingly
- Load, parse, and look up character voice chat files (`.voice`, `.vc`)
- Buffer and throttle voice chat playback with a ring buffer
- Handle map restarts, warmup transitions, and shader remapping

## External Dependencies
- `cg_local.h` — all cgame types (`cg_t`, `cgs_t`, `clientInfo_t`, trap functions, cvars)
- `ui/menudef.h` — `VOICECHAT_*` string constants and UI owner-draw defines
- **Defined elsewhere:** `CG_ConfigString`, `CG_Argv`, `CG_StartMusic`, `CG_NewClientInfo`, `CG_BuildSpectatorString`, `CG_InitLocalEntities`, `CG_InitMarkPolys`, `CG_ClearParticles`, `CG_SetScoreSelection`, `CG_ShowResponseHead`, `CG_LoadDeferredPlayers`, `COM_ParseExt`, `Info_ValueForKey`, all `trap_*` syscalls

# code/cgame/cg_snapshot.c
## File Purpose
Manages the client-side snapshot pipeline, advancing the simulation clock by transitioning between server-delivered game state snapshots. It handles initial snapshot setup, interpolation state tracking, entity transitions, and teleport detection — all without necessarily firing every rendered frame.

## Core Responsibilities
- Read new snapshots from the client system into a double-buffered slot
- Initialize all entity state on the very first snapshot (or map restart)
- Transition `cg.nextSnap` → `cg.snap` when simulation time crosses the boundary
- Set `centity_t.interpolate` flags so the renderer knows whether to lerp or snap entities
- Detect teleport events (both entity-level and playerstate-level) and suppress interpolation accordingly
- Fire entity and playerstate events during snapshot transitions
- Record lagometer data for dropped/received snapshots

## External Dependencies

- **Includes:** `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere:**
  - `trap_GetCurrentSnapshotNumber`, `trap_GetSnapshot` — client system traps
  - `BG_PlayerStateToEntityState` — `bg_misc.c`
  - `CG_BuildSolidList` — `cg_predict.c`
  - `CG_ExecuteNewServerCommands` — `cg_servercmds.c`
  - `CG_Respawn`, `CG_TransitionPlayerState` — `cg_playerstate.c`
  - `CG_CheckEvents` — `cg_events.c`
  - `CG_ResetPlayerEntity` — `cg_players.c`
  - `CG_AddLagometerSnapshotInfo` — `cg_draw.c`
  - `cg`, `cgs`, `cg_entities` — `cg_main.c`

# code/cgame/cg_syscalls.c
## File Purpose
Implements the cgame module's system call interface for the DLL build path. Each `trap_*` function wraps a variadic `syscall` function pointer that dispatches into the engine using integer opcode identifiers defined in `cg_public.h`.

## Core Responsibilities
- Receive and store the engine-provided syscall dispatcher via `dllEntry`
- Expose typed `trap_*` wrappers for every engine service the cgame module needs
- Convert `float` arguments to `int`-width bit-reinterpretations via `PASSFLOAT` before passing through the integer-only syscall ABI
- Cover all engine subsystems: console, cvar, filesystem, collision, sound, renderer, input, cinematic, and snapshot/game-state retrieval

## External Dependencies
- **Includes:** `cg_local.h` → transitively pulls in `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:** All `CG_*` opcode constants (e.g., `CG_PRINT`, `CG_R_RENDERSCENE`) — defined in `cg_public.h`; all struct types (`trace_t`, `refEntity_t`, `snapshot_t`, `glconfig_t`, etc.) — defined in shared/renderer headers; `QDECL` calling-convention macro — from `q_shared.h`

# code/cgame/cg_view.c
## File Purpose
Sets up all 3D rendering parameters (view origin, view angles, FOV, viewport rect) each frame and issues the final render call. It is the central per-frame orchestration point for the cgame's visual output.

## Core Responsibilities
- Compute viewport rectangle based on `cg_viewsize` cvar
- Offset first-person or third-person view with bobbing, damage kick, step smoothing, duck smoothing, and land bounce
- Calculate FOV with zoom interpolation and underwater warp
- Build and submit the `refdef_t` to the renderer via `CG_DrawActive`
- Add all scene entities (packet entities, marks, particles, local entities, view weapon, test model)
- Manage a circular sound buffer for announcer/sequential sounds
- Emit powerup-expiry warning sounds
- Provide developer model-testing commands (`testmodel`, `testgun`, frame/skin cycling)

## External Dependencies
- `cg_local.h` — all cgame types, `cg`, `cgs`, cvars, trap declarations
- `trap_R_*` — renderer scene API (defined in engine)
- `trap_S_*` — sound API (defined in engine)
- `CG_DrawActive` — defined in `cg_draw.c`
- `CG_PredictPlayerState`, `CG_Trace`, `CG_PointContents` — defined in `cg_predict.c`
- `CG_AddPacketEntities` — defined in `cg_ents.c`
- `CG_AddViewWeapon` — defined in `cg_weapons.c`
- `CG_PlayBufferedVoiceChats` — defined in `cg_servercmds.c`
- `AnglesToAxis`, `VectorMA`, `DotProduct`, `AngleVectors` — math utilities from `q_math.c`

# code/cgame/cg_weapons.c
## File Purpose
Client-side weapon visualization module for Quake III Arena. Handles all weapon-related rendering, effects, and input, including view weapon display, projectile trails, muzzle flashes, impact effects, shell ejection, and weapon selection UI.

## Core Responsibilities
- Register and cache weapon/item media (models, shaders, sounds) at level load
- Render the first-person view weapon with bobbing, FOV offset, and animation mapping
- Render world-space weapon models attached to player entities (with powerup overlays)
- Emit per-weapon trail effects (rocket smoke, rail rings, plasma sparks, grapple beam)
- Spawn muzzle flash, dynamic light, and brass ejection local entities on fire events
- Resolve hitscan impact effects (explosions, marks, sounds) for all weapon types
- Simulate shotgun pellet spread client-side (matching server seed) for decals/sounds
- Manage weapon cycling (next/prev/direct select) and on-screen weapon selection HUD

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `BG_EvaluateTrajectory` — defined in `bg_misc.c`
- `CG_AllocLocalEntity`, `CG_SmokePuff`, `CG_MakeExplosion`, `CG_BubbleTrail`, `CG_Bleed`, `CG_ImpactMark`, `CG_ParticleExplosion` — defined in other cgame modules
- `CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag` — `cg_ents.c`
- `trap_R_*`, `trap_S_*`, `trap_CM_*` — VM syscall stubs (`cg_syscalls.c`)
- `axisDefault`, `vec3_origin` — defined in `q_math.c` / `q_shared.c`

# code/cgame/tr_types.h
## File Purpose
Defines the shared renderer interface types used by both the client-game (cgame) module and the renderer. It establishes the data structures and constants that describe renderable entities, scene definitions, and OpenGL hardware configuration.

## Core Responsibilities
- Define render entity types and the `refEntity_t` descriptor passed to the renderer
- Define `refdef_t`, the per-frame scene/camera description
- Define OpenGL capability and configuration types (`glconfig_t`)
- Declare bit-flag constants for render effects (`RF_*`) and render definition flags (`RDF_*`)
- Establish hard limits on dynamic lights and renderable entities
- Define polygon vertex and polygon types for decal/effect geometry

## External Dependencies
- **Defined elsewhere:** `vec3_t`, `qhandle_t`, `qboolean`, `byte`, `MAX_STRING_CHARS`, `BIG_INFO_STRING`, `MAX_MAP_AREA_BYTES` — all from `q_shared.h`
- Driver name macros (`_3DFX_DRIVER_NAME`, `OPENGL_DRIVER_NAME`) conditionalized on `Q3_VM` and `_WIN32` platform defines
- `MAX_DLIGHTS 32` is a hard architectural limit because dlight influence is stored as a 32-bit surface bitmask; `MAX_ENTITIES 1023` is constrained by drawsurf sort-key bit packing in the renderer

# code/client/cl_cgame.c
## File Purpose
This file implements the client-side interface layer between the engine and the cgame VM module. It provides the system call dispatch table that the cgame VM invokes to access engine services, and manages cgame VM lifecycle (init, shutdown, per-frame rendering and time updates).

## Core Responsibilities
- Load, initialize, and shut down the cgame VM (`VM_Create`/`VM_Free`)
- Dispatch all cgame system calls (`CL_CgameSystemCalls`) to appropriate engine subsystems
- Expose client state to cgame: snapshots, user commands, game state, GL config, server commands
- Process server commands destined for cgame (`CL_GetServerCommand`) including large config string reassembly (`bcs0/bcs1/bcs2`)
- Manage configstring updates (`CL_ConfigstringModified`) into `cl.gameState`
- Drive server time synchronization and drift correction (`CL_SetCGameTime`, `CL_AdjustTimeDelta`)
- Trigger cgame rendering each frame (`CL_CGameRendering`)

## External Dependencies
- `client.h` → pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `cg_public.h`, `bg_public.h`, `keys.h`, `snd_public.h`
- `botlib.h` — `botlib_export_t *botlib_export` (defined in `be_interface.c`)
- `cgvm` — `vm_t *` defined in `cl_main.c`
- `re` — `refexport_t` renderer interface (defined in `cl_main.c`)
- Camera functions (`loadCamera`, `startCamera`, `getCameraInfo`) — declared extern, all call sites commented out
- `CM_*`, `S_*`, `FS_*`, `Key_*`, `CIN_*`, `Cbuf_*`, `Cvar_*`, `Cmd_*`, `Hunk_*`, `Sys_*`, `Com_*` — all defined elsewhere in engine subsystems

# code/client/cl_cin.c
## File Purpose
Implements RoQ video cinematic playback for Quake III Arena, handling decoding of RoQ-format video frames (VQ-compressed), YUV-to-RGB color conversion, audio decompression (RLL-encoded mono/stereo), and rendering of cinematics to the screen or in-game surfaces.

## Core Responsibilities
- Parse and decode RoQ video file format (header, codebook, VQ frames, audio packets)
- Perform YUV→RGB(16-bit and 32-bit) color space conversion using precomputed lookup tables
- Decode RLL-encoded audio (mono/stereo variants) into PCM samples and feed to the sound system
- Manage up to 16 simultaneous video handles (`cinTable[MAX_VIDEO_HANDLES]`)
- Build and cache the quad-tree blitting structure for VQ frame rendering
- Handle looping, hold-at-end, in-game shader video, and game-state transitions
- Upload decoded frames to the renderer via `re.DrawStretchRaw` / `re.UploadCinematic`

## External Dependencies
- `client.h`: `cls`, `cl`, `uivm`, `re` (renderer), `com_timescale`, `cl_inGameVideo`, `SCR_AdjustFrom640`, `CL_ScaledMilliseconds`
- `snd_local.h`: `s_rawend`, `s_soundtime`, `s_paintedtime`
- Sound: `S_RawSamples`, `S_Update`, `S_StopAllSounds`
- Filesystem: `FS_FOpenFileRead`, `FS_FCloseFile`, `FS_Read`
- Streaming I/O: `Sys_BeginStreamedFile`, `Sys_EndStreamedFile`, `Sys_StreamedRead`
- Renderer: `re.DrawStretchRaw`, `re.UploadCinematic`
- Memory: `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`
- `glConfig.hardwareType`, `glConfig.maxTextureSize` — hardware capability checks

# code/client/cl_console.c
## File Purpose
Implements the in-game developer console for Quake III Arena, handling text buffering, scrollback, notify overlays, animated slide-in/out drawing, and chat message input modes.

## Core Responsibilities
- Maintain a circular text buffer (`con.text`) for scrollback history
- Handle line wrapping, word wrapping, and color-coded character storage
- Animate console slide open/close via `displayFrac`/`finalFrac` interpolation
- Render the solid console panel, scrollback arrows, version string, and input prompt
- Render transparent notify lines (recent messages) over the game view
- Manage chat input modes (global, team, crosshair target, last attacker)
- Register console-related commands (`toggleconsole`, `clear`, `condump`, etc.)

## External Dependencies
- `client.h` → pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `keys.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `cls` (`clientStatic_t`), `cl` (`clientActive_t`), `cgvm`, `re` (renderer exports), `g_consoleField`, `chatField`, `chat_playerNum`, `chat_team`, `historyEditLines`, `g_color_table`, `cl_noprint`, `cl_conXOffset`, `com_cl_running`; renderer entry points `SCR_DrawSmallChar`, `SCR_DrawPic`, `SCR_FillRect`, `Field_Draw`, `Field_BigDraw`, `Field_Clear`, `VM_Call`

# code/client/cl_input.c
## File Purpose
Translates raw input events (keyboard, mouse, joystick) into `usercmd_t` structures and transmits them to the server each frame. It manages continuous button state tracking and builds the outgoing command packet.

## Core Responsibilities
- Track key/button press and release state via `kbutton_t`, supporting two simultaneous keys per logical button
- Convert key states into fractional movement values scaled by frame time
- Adjust view angles from keyboard and joystick inputs
- Accumulate mouse delta into view angle or movement changes
- Assemble `usercmd_t` per frame from all input sources
- Rate-limit outgoing packets via `cl_maxpackets` and `CL_ReadyToSendPacket`
- Serialize and transmit the command packet with delta-compressed usercmds

## External Dependencies
- `client.h` — brings in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `cl`, `clc`, `cls` (global client state structs); `com_frameTime`; `anykeydown`; `cl_sensitivity`, `cl_mouseAccel`, `cl_freelook`, `cl_showMouseRate`, `m_pitch`, `m_yaw`, `m_forward`, `m_side`, `m_filter`, `cl_maxpackets`, `cl_packetdup`, `cl_showSend`, `cl_nodelta`, `sv_paused`, `cl_paused`, `com_sv_running`; `VM_Call`; `uivm`, `cgvm`; `Cmd_Argv`, `Cmd_AddCommand`; `Cvar_Get`, `Cvar_Set`; `MSG_*` family; `CL_Netchan_Transmit`; `SCR_DebugGraph`; `ClampChar`, `VectorCopy`, `SHORT2ANGLE`, `ANGLE2SHORT`; `Sys_IsLANAddress`; `Com_HashKey`; `IN_CenterView`.

# code/client/cl_keys.c
## File Purpose
Implements the client-side keyboard input system for Quake III Arena, managing key bindings, key state tracking, text field editing (console/chat), and dispatching input events to the appropriate subsystem (console, UI VM, cgame VM, or game commands).

## Core Responsibilities
- Maintain the `keys[]` array of key states (down, repeats, binding)
- Translate between key name strings and key numbers (bidirectionally)
- Handle console field and chat field line editing (cursor, scrolling, history)
- Dispatch key-down/key-up events to the correct handler based on `cls.keyCatchers`
- Execute bound commands (immediate and `+button` style with up/down pairing)
- Register `bind`, `unbind`, `unbindall`, `bindlist` console commands
- Write key bindings to config files

## External Dependencies
- **Includes:** `client.h` → `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- **Defined elsewhere:** `Field_Clear`, `Field_CompleteCommand` (likely `cl_console.c`); `Con_PageUp/Down/Top/Bottom/ToggleConsole_f` (`cl_console.c`); `VM_Call` (`vm.c`); `Cbuf_AddText`, `Cmd_AddCommand`, `Cmd_Argc/Argv` (`cmd.c`); `Cvar_Set/VariableValue` (`cvar.c`); `Z_Free`, `CopyString` (memory); `FS_Printf` (`files.c`); `Sys_GetClipboardData` (platform); `SCR_Draw*` (`cl_scrn.c`); `CL_AddReliableCommand` (`cl_main.c`); `cvar_modifiedFlags` (`cvar.c`); `cls`, `clc`, `cgvm`, `uivm` (client globals).

# code/client/cl_main.c
## File Purpose
This is the central client subsystem manager for Quake III Arena, responsible for initializing, running, and shutting down all client-side systems. It drives the per-frame client loop, manages the connection state machine (connecting → challenging → connected → active), and owns demo recording/playback, server discovery, and reliable command queuing.

## Core Responsibilities
- Register and manage all client-side cvars and console commands
- Drive the per-frame `CL_Frame` loop: input, timeout, packet send, screen/audio/cinematic update
- Manage connection lifecycle: connect, disconnect, reconnect, challenge/authorize handshake
- Record and play back demo files (write gamestate snapshot + replayed net messages)
- Handle out-of-band connectionless packets (challenge, MOTD, server status, server list)
- Manage file downloads from server (download queue, temp files, FS restart on completion)
- Initialize and teardown renderer, sound, UI, and cgame subsystems via hunk lifecycle
- Provide server browser ping infrastructure (`cl_pinglist`, `CL_UpdateVisiblePings_f`)

## External Dependencies
- **Includes:** `client.h` (aggregates `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`); `<limits.h>`
- **Defined elsewhere:**
  - `GetRefAPI` — renderer module export
  - `SV_BotFrame`, `SV_Shutdown`, `SV_Frame` — server module
  - `CL_ParseServerMessage` — `cl_parse.c`
  - `CL_SendCmd`, `CL_WritePacket`, `CL_InitInput` — `cl_input.c`
  - `CL_InitCGame`, `CL_ShutdownCGame`, `CL_SetCGameTime` — `cl_cgame.c`
  - `CL_InitUI`, `CL_ShutdownUI` — `cl_ui.c`
  - `CL_Netchan_Process` — `cl_net_chan.c`
  - `S_Init`, `S_Shutdown`, `S_Update`, `S_DisableSounds`, etc. — sound subsystem
  - `Hunk_*`, `CM_*`, `FS_*`, `NET_*`, `Cvar_*`, `Cmd_*`, `MSG_*` — `qcommon`

# code/client/cl_net_chan.c
## File Purpose
Provides the client-side network channel layer, wrapping the core `Netchan_*` functions with client-specific XOR obfuscation for outgoing and incoming game packets. It encodes transmitted messages and decodes received messages using a rolling key derived from the client challenge, server/sequence IDs, and acknowledged command strings.

## Core Responsibilities
- Encode outgoing client messages (bytes after `CL_ENCODE_START`) before transmission
- Decode incoming server messages (bytes after `CL_DECODE_START`) after reception
- Append `clc_EOF` marker before encoding and transmitting
- Delegate fragment transmission to the base `Netchan_TransmitNextFragment`
- Accumulate decoded byte counts in `newsize` for diagnostics/comparison with `oldsize`

## External Dependencies
- `../game/q_shared.h` — base types (`byte`, `qboolean`, `msg_t` fields)
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `Netchan_Transmit`, `Netchan_TransmitNextFragment`, `Netchan_Process`, `MSG_ReadLong`, `MSG_WriteByte`, `LittleLong`, `CL_ENCODE_START`, `CL_DECODE_START`, `MAX_RELIABLE_COMMANDS`, `clc_EOF`
- `client.h` — `clc` (`clientConnection_t`: `challenge`, `serverCommands`, `reliableCommands`)
- `oldsize` — `extern int` defined elsewhere (likely `cl_parse.c`) used for bandwidth comparison

# code/client/cl_parse.c
## File Purpose
Parses incoming server-to-client network messages for Quake III Arena. It decodes the server message stream into snapshots, entity states, game state, downloads, and server commands that the client uses to update its local world representation.

## Core Responsibilities
- Dispatch incoming server messages by opcode (`svc_*`)
- Parse full game state on level load/connection (configstrings + entity baselines)
- Parse delta-compressed snapshots (player state + packet entities)
- Reconstruct entity states via delta decompression from prior frames or baselines
- Handle file download protocol (block-based chunked transfer)
- Store server command strings for deferred cgame execution
- Sync client-side cvars from server `systeminfo` configstring

## External Dependencies
- **Includes:** `client.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`)
- **Defined elsewhere:** `cl` (`clientActive_t`), `clc` (`clientConnection_t`), `cls` (`clientStatic_t`), `cl_shownet` (cvar), `MSG_Read*` family (msg.c), `FS_*` (files.c), `Cvar_*` (cvar.c), `CL_AddReliableCommand` / `CL_WritePacket` / `CL_NextDownload` / `CL_ClearState` / `CL_InitDownloads` (cl_main.c), `Con_Close` (console), `Info_*` (q_shared.c)

# code/client/cl_scrn.c
## File Purpose
Manages the screen rendering pipeline for the Quake III Arena client, orchestrating the drawing of all 2D screen elements (HUD, console, debug graphs, demo recording indicator) and driving the per-frame refresh cycle. It also provides a set of virtual-resolution drawing utilities used throughout the client and UI code.

## Core Responsibilities
- Initialize screen-related CVars and set the `scr_initialized` flag
- Convert 640×480 virtual coordinates to actual screen resolution
- Draw 2D primitives: filled rectangles, named/handle-based shaders, big/small chars and strings with color codes
- Drive the per-frame screen update, handling stereo rendering and speed profiling
- Dispatch rendering to the appropriate subsystem based on connection state (cinematic, loading, active game, menus)
- Maintain and render the debug/timing graph overlay

## External Dependencies
- **Includes:** `client.h` (transitively pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `cg_public.h`, `bg_public.h`, `keys.h`, `snd_public.h`)
- **Defined elsewhere:** `re` (`refexport_t`), `cls` (`clientStatic_t`), `clc` (`clientConnection_t`), `uivm` (`vm_t *`), `g_color_table`, `com_speeds`, `time_frontend`, `time_backend`, `cl_debugMove`, `VM_Call`, `Con_DrawConsole`, `CL_CGameRendering`, `SCR_DrawCinematic`, `S_StopAllSounds`, `FS_FTell`, `Com_Error`, `Com_DPrintf`, `Com_Memcpy`, `Q_IsColorString`, `ColorIndex`, `Cvar_Get`

# code/client/cl_ui.c
## File Purpose
This file implements the client-side UI virtual machine (VM) bridge layer, providing the system call dispatch table that translates UI module requests into engine function calls. It also manages the UI VM lifecycle (init/shutdown) and maintains the server browser (LAN) data structures with cache persistence.

## Core Responsibilities
- Dispatch all `UI_*` system calls from the UI VM to engine subsystems via `CL_UISystemCalls`
- Initialize and shut down the UI VM (`CL_InitUI`, `CL_ShutdownUI`)
- Provide LAN server list management: add, remove, query, compare, and mark visibility across four server sources (local, mplayer, global, favorites)
- Persist and restore server browser caches to/from `servercache.dat`
- Bridge UI requests to renderer (`re.*`), sound (`S_*`), key system, filesystem, cinematic, and botlib parse contexts
- Expose client/connection state (`GetClientState`, `CL_GetGlconfig`) and config strings to the UI VM

## External Dependencies
- **Includes:** `client.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`), `../game/botlib.h`
- **Defined elsewhere:** `cls` (`clientStatic_t`), `clc` (`clientConnection_t`), `cl` (`clientActive_t`), `re` (`refexport_t`), `cl_connectedToPureServer`, `cl_cdkey`, `cvar_modifiedFlags`, `VM_Create/Call/Free/ArgPtr`, `NET_*`, `FS_*`, `S_*`, `Key_*`, `CIN_*`, `SCR_UpdateScreen`, `Sys_GetClipboardData`, `Sys_Milliseconds`, `Hunk_MemoryRemaining`, `Com_RealTime`, `CL_CDKeyValidate`, `CL_ServerStatus`, `CL_UpdateVisiblePings_f`, `CL_GetPing*`, `Z_Free`

# code/client/client.h
## File Purpose
Primary header for the Quake III Arena client subsystem. Defines the three core client state structures (`clSnapshot_t`, `clientActive_t`, `clientConnection_t`, `clientStatic_t`) and declares all function prototypes for every client-side module: main, input, parsing, console, screen, cinematics, cgame, UI, and network channel.

## Core Responsibilities
- Defines snapshot representation for server-to-client delta-compressed state
- Defines the three-tier client state hierarchy (active/connection/static)
- Declares all inter-module function interfaces for the client subsystem
- Declares the global extern instances (`cl`, `clc`, `cls`) shared across client modules
- Declares VM handles for cgame and UI modules (`cgvm`, `uivm`, `re`)
- Declares all client-facing cvars as extern pointers
- Pulls in all required subsystem headers (renderer, UI, sound, cgame, shared)

## External Dependencies
- `../game/q_shared.h` — shared math, entity/player state types, `qboolean`, `cvar_t`
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `vm_t`, filesystem, `netadr_t`, `connstate_t`
- `../renderer/tr_public.h` — `refexport_t`, `glconfig_t`, `stereoFrame_t`
- `../ui/ui_public.h` — `uiClientState_t`, `uiMenuCommand_t`
- `keys.h` — `qkey_t`, key binding declarations, `field_t` input fields
- `snd_public.h` — sound system public interface (included but not shown)
- `../cgame/cg_public.h` — cgame public interface, `stereoFrame_t`
- `../game/bg_public.h` — `playerState_t`, `usercmd_t`, pmove shared definitions
- **Defined elsewhere:** `vm_t`, `gameState_t`, `entityState_t`, `playerState_t`, `usercmd_t`, `netchan_t`, all `Netchan_*` base functions, `MSG_*` functions

# code/client/keys.h
## File Purpose
Declares the key input subsystem interface for the Quake III Arena client, defining key state storage, text input field operations, and the public API for key binding management.

## Core Responsibilities
- Defines the `qkey_t` struct representing per-key state (down/repeat/binding)
- Declares the global `keys[MAX_KEYS]` array as the central key state table
- Exposes text input field rendering and event functions for console/chat UI
- Declares the command history ring buffer and active console/chat fields
- Provides the public API for reading, writing, and querying key bindings
- Exposes insert/overstrike mode toggle state

## External Dependencies
- `../ui/keycodes.h` — defines `keyNum_t` enum covering all 256 possible key slots
- `field_t` — declared in `qcommon/qcommon.h` (noted inline by TTimo)
- `fileHandle_t` — defined in `q_shared.h` / `qcommon.h`
- `qboolean` — defined in `q_shared.h`

# code/client/snd_adpcm.c
## File Purpose
Implements Intel/DVI ADPCM (Adaptive Differential Pulse-Code Modulation) audio compression and decompression for Quake III Arena's sound system. It encodes raw PCM audio into a 4-bit-per-sample ADPCM format and decodes it back, and provides the glue functions to store/retrieve ADPCM-compressed sound data in the engine's chunked `sndBuffer` system.

## Core Responsibilities
- Encode 16-bit PCM samples into 4-bit ADPCM nibbles (`S_AdpcmEncode`)
- Decode 4-bit ADPCM nibbles back to 16-bit PCM samples (`S_AdpcmDecode`)
- Calculate memory requirements for ADPCM-compressed sound assets (`S_AdpcmMemoryNeeded`)
- Retrieve decoded samples from a single `sndBuffer` chunk (`S_AdpcmGetSamples`)
- Encode an entire `sfx_t` sound asset into a linked list of `sndBuffer` chunks (`S_AdpcmEncodeSound`)

## External Dependencies
- **Includes:** `snd_local.h` → pulls in `q_shared.h`, `qcommon.h`, `snd_public.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`) — global DMA state providing `dma.speed`
  - `SND_malloc()` — sndBuffer allocator (defined in `snd_mem.c`)
  - `PAINTBUFFER_SIZE`, `SND_CHUNK_SIZE_BYTE` — macros from `snd_local.h`
  - `adpcm_state_t`, `sndBuffer`, `sfx_t`, `wavinfo_t` — types from `snd_local.h`

# code/client/snd_dma.c
## File Purpose
Main control module for the Quake III Arena software-mixed sound system. It manages sound channel allocation, spatialization, looping sounds, background music streaming, and drives the DMA mixing pipeline each frame.

## Core Responsibilities
- Initialize and shut down the sound system via `SNDDMA_*` platform layer
- Register, cache, and evict sound assets (`sfx_t`) from memory
- Allocate and manage `channel_t` slots for one-shot and looping sounds
- Spatialize 3D sound channels using listener position and orientation
- Stream background music from WAV files into the raw sample buffer
- Drive the mixing pipeline (`S_PaintChannels`) each frame via `S_Update_`
- Handle Doppler scaling for looping sounds tied to moving entities

## External Dependencies
- `snd_local.h`: `sfx_t`, `channel_t`, `dma_t`, `loopSound_t`, `SNDDMA_*`, `S_PaintChannels`, `SND_malloc/free`, `S_LoadSound`
- `client.h`: `cls.framecount` (Doppler frame tracking)
- **Defined elsewhere**: `SNDDMA_Init/Shutdown/GetDMAPos/BeginPainting/Submit` (platform layer: `win_snd.c` / `linux_snd.c`), `S_PaintChannels` (`snd_mix.c`), `S_LoadSound` (`snd_mem.c`), `Sys_BeginStreamedFile/StreamedRead/EndStreamedFile` (OS layer), `VectorRotate`, `DistanceSquared` (math), `Com_Milliseconds`, `FS_Read/FOpenFileRead/FCloseFile`

# code/client/snd_local.h
## File Purpose
Private internal header for Quake III Arena's software sound mixing system. It defines all core data structures, buffer layouts, global state declarations, and internal function prototypes used across the sound subsystem's mixing, spatialization, ADPCM compression, and wavelet/mu-law encoding modules.

## Core Responsibilities
- Define sample buffer structures (`sndBuffer`, `portable_samplepair_t`) for the mixing pipeline
- Define the `sfx_t` sound effect asset type with optional compression metadata
- Define `channel_t` for active playback channels with spatialization state
- Define `dma_t` describing the platform DMA output buffer
- Declare all cross-module globals (channels, listener orientation, cvars, raw sample buffer)
- Declare internal API for sound loading, mixing, spatialization, ADPCM, and wavelet codec functions
- Declare platform-abstraction stubs (`SNDDMA_*`) that must be implemented per OS

## External Dependencies
- `q_shared.h` — `vec3_t`, `qboolean`, `cvar_t`, `byte`, `MAX_QPATH`
- `qcommon.h` — `Z_Malloc`/`S_Malloc`, `Cvar_Get`, `FS_ReadFile`, `Com_Printf`
- `snd_public.h` — public sound API declarations consumed by client layer
- `SNDDMA_*` functions — defined elsewhere in platform-specific files (`win_snd.c`, `linux_snd.c`, `snd_null.c`)
- `mulawToShort[]` — defined in `snd_adpcm.c` or `snd_wavelet.c`

# code/client/snd_mem.c
## File Purpose
Implements the sound memory manager and WAV file loader for Quake III Arena's audio system. It manages a fixed-size pool of `sndBuffer` chunks via a free-list allocator, parses WAV headers, and resamples raw PCM audio to match the engine's DMA output rate.

## Core Responsibilities
- Initialize and manage a slab-based free-list allocator for `sndBuffer` chunks
- Parse RIFF/WAV file headers to extract format metadata (`wavinfo_t`)
- Resample PCM audio data (8-bit or 16-bit, mono) from source rate to `dma.speed`
- Load and decode sound assets into `sfx_t` structures, optionally applying ADPCM compression
- Report free/used sound memory statistics

## External Dependencies
- **Includes:** `snd_local.h` → `q_shared.h`, `qcommon.h`, `snd_public.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`) — global DMA state; `dma.speed` used for resampling ratio
  - `S_FreeOldestSound` — eviction policy, defined in `snd_dma.c`
  - `S_AdpcmEncodeSound` — ADPCM encoder, defined in `snd_adpcm.c`
  - `LittleShort` — endian swap macro, from `q_shared.h`
  - `FS_ReadFile`, `FS_FreeFile`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Com_Milliseconds`, `Cvar_Get`, `Com_Printf`, `Com_DPrintf`, `Com_Memset` — engine common layer

# code/client/snd_mix.c
## File Purpose
Implements the portable audio mixing pipeline for Quake III Arena's DMA sound system. It reads from active sound channels, mixes them into an intermediate paint buffer, and transfers the result into the platform DMA output buffer.

## Core Responsibilities
- Maintain and fill the stereo `paintbuffer` intermediate mix buffer
- Mix one-shot and looping sound channels into the paint buffer per-frame
- Support four audio decompression paths: raw PCM 16-bit, ADPCM, Wavelet, and Mu-Law
- Apply volume scaling and optional Doppler pitch shifting during mixing
- Transfer the paint buffer to the DMA output buffer with bit-depth/channel-count adaptation
- Provide platform-specific fast paths: x86 inline asm (`id386`) and AltiVec SIMD (`idppc_altivec`)

## External Dependencies
- **`snd_local.h`** — all shared types, channel arrays, DMA state, cvars, and scratch buffer globals
- **`s_channels[MAX_CHANNELS]`**, **`loop_channels`**, **`numLoopChannels`** — defined in `snd_dma.c`
- **`s_paintedtime`**, **`s_rawend`**, **`s_rawsamples`**, **`dma`** — defined in `snd_dma.c`
- **`s_volume`**, **`s_testsound`** — cvars registered in `snd_dma.c`
- **`sfxScratchBuffer`**, **`sfxScratchPointer`**, **`sfxScratchIndex`** — defined in `snd_mem.c`
- **`mulawToShort[256]`** — lookup table defined in `snd_adpcm.c`
- **`S_AdpcmGetSamples`**, **`decodeWavelet`** — defined in `snd_adpcm.c` / `snd_wavelet.c`
- **`S_WriteLinearBlastStereo16`** (Linux x86) — implemented in `unix/snd_mixa.s`
- **`Com_Memset`** — defined in `qcommon`

# code/client/snd_public.h
## File Purpose
Public interface header for the Quake III Arena sound system, exposing all externally callable sound functions to other engine subsystems (client, cgame, etc.). It declares the full lifecycle API for sound playback, looping sounds, spatialization, and background music.

## Core Responsibilities
- Declare sound system initialization and shutdown entry points
- Expose one-shot and looping 3D spatialized sound playback functions
- Provide background music track control (intro + loop)
- Declare raw PCM sample injection for cinematics and VoIP
- Define entity-based position update and reverberation/spatialization calls
- Expose sound registration (asset loading) interface
- Provide utility/diagnostic functions (free memory display, buffer clearing)

## External Dependencies
- `vec3_t`, `qboolean`, `byte` — defined in `q_shared.h`
- `sfxHandle_t` — defined in `q_shared.h` or `snd_local.h`
- All function bodies defined in `snd_dma.c`, `snd_mix.c`, `snd_mem.c` (and platform DMA backends)

# code/client/snd_wavelet.c
## File Purpose
Implements wavelet-based and mu-law audio compression/decompression for Quake III's sound system. It encodes PCM audio data into compact `sndBuffer` chunks using either a Daubechies-4 wavelet transform followed by mu-law quantization, or mu-law encoding alone with dithered error feedback.

## Core Responsibilities
- Apply forward/inverse Daubechies-4 (daub4) wavelet transform to float sample arrays
- Drive multi-resolution wavelet decomposition/reconstruction via `wt1`
- Encode 16-bit PCM samples to 8-bit mu-law bytes (`MuLawEncode`)
- Decode 8-bit mu-law bytes back to 16-bit PCM (`MuLawDecode`)
- Build and cache the `mulawToShort[256]` lookup table on first use
- Compress an `sfx_t` sound asset into linked `sndBuffer` chunks (`encodeWavelet`, `encodeMuLaw`)
- Decompress `sndBuffer` chunks back to PCM for mixing (`decodeWavelet`, `decodeMuLaw`)

## External Dependencies
- `snd_local.h` — `sfx_t`, `sndBuffer`, `SND_CHUNK_SIZE`, `SND_malloc`, `NXStream`, `qboolean`, `byte`, `short`
- `SND_malloc` — defined in `snd_mem.c`
- `myftol` — declared but not called in this file; defined elsewhere (platform float-to-long helper)
- `numBits[256]` — file-static lookup table for bit-count of byte values

# code/game/ai_chat.c
## File Purpose
Implements the bot chat/taunting AI layer for Quake III Arena. It decides when and what a bot says in response to game events (entering/exiting a game, kills, deaths, random chatter, etc.), gating all output behind cooldown timers, game-mode checks, and bot personality characteristics.

## Core Responsibilities
- Enforce chat rate-limiting via `TIME_BETWEENCHATTING` (25 s cooldown)
- Query player rankings and opponent lists to populate chat template variables
- Validate whether a bot is in a safe position to chat (not in lava/water, on solid ground, no active powerups, no visible enemies)
- Select the appropriate chat category string (e.g., `"death_rail"`, `"kill_insult"`) based on game context and random characteristic weights
- Delegate actual message construction and queuing to `BotAI_BotInitialChat` / `trap_BotEnterChat`
- Issue `vtaunt` voice commands in team-play modes instead of text chat
- Provide `BotChatTest` to exhaustively exercise all chat categories for debugging

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_char.h`, `be_ai_chat.h`, `be_ai_gen.h`, `be_ai_goal.h`, `be_ai_move.h`, `be_ai_weap.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`; conditionally `ui/menudef.h` (MissionPack)
- **Defined elsewhere:** `bot_state_t`, `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotAI_Trace`, `EasyClientName`, `ClientName`, `BotEntityInfo`, `BotEntityVisible`, `BotSameTeam`, `BotIsDead`, `BotIsObserver`, `EntityIsDead`, `EntityIsInvisible`, `EntityIsShooting`, `TeamPlayIsOn`, `FloatTime`, `gametype`, `bot_nochat`, `bot_fastchat`, `g_entities`, all `trap_*` syscalls

# code/game/ai_chat.h
## File Purpose
Public interface header declaring bot AI chat functions for Quake III Arena. It exposes event-driven chat triggers and utility functions that allow bots to send contextually appropriate chat messages during gameplay.

## Core Responsibilities
- Declare chat event hooks for game lifecycle events (enter/exit game, level start/end)
- Declare combat-contextual chat triggers (hit, death, kill, suicide)
- Declare utility functions for chat timing, position validation, and testing

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` / `g_local.h` (defined elsewhere)
- Implementation resides in `code/game/ai_chat.c`

# code/game/ai_cmd.c
## File Purpose
Implements the bot AI command-processing layer for Quake III Arena's team-play modes. It parses structured natural-language chat matches (e.g. "help me", "defend the flag") received from human players and translates them into long-term goal (LTG) state changes on the receiving `bot_state_t`. It is the bridge between the bot chat-matching subsystem and the bot goal/behavior system.

## Core Responsibilities
- Receive a raw chat string via `BotMatchMessage`, classify it against known message templates (`trap_BotFindMatch`), and dispatch to a typed handler.
- Determine whether a match message is actually addressed to this bot (`BotAddressedToBot`).
- Resolve named teammates, enemies, map items, and waypoints from human-readable strings into engine-usable identifiers.
- Set `bs->ltgtype`, `bs->teamgoal`, `bs->teamgoal_time`, and related fields on the bot state to steer high-level behavior.
- Manage bot sub-team membership, team-leader tracking, and the `notleader[]` flag array.
- Parse and store patrol waypoint chains and user-defined checkpoint waypoints.
- Track CTF/1FCTF flag status changes reported through team chat.

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_char/chat/gen/goal/move/weap.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`, `ui/menudef.h`
- **Defined elsewhere (used here):** `bot_state_t`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`, `BotGetAlternateRouteGoal`, `BotOppositeTeam`, `BotSameTeam`, `BotTeam`, `BotFindWayPoint`, `BotCreateWayPoint`, `BotFreeWaypoints`, `BotVoiceChat`, `BotVoiceChatOnly`, `TeamPlayIsOn`, `ClientFromName`, `ClientOnSameTeamFromName`, `EasyClientName`, `BotAI_BotInitialChat`, `BotAI_Trace`, `FloatTime`, `gametype`, `ctf_redflag`, `ctf_blueflag`, all `trap_*` syscalls.

# code/game/ai_cmd.h
## File Purpose
Header file for the bot AI command/message processing subsystem in Quake III Arena. It declares the public interface for bot team-command parsing and team goal reporting used by the game module's AI layer.

## Core Responsibilities
- Exposes the `BotMatchMessage` function for parsing and dispatching incoming chat/voice commands to a bot
- Exposes `BotPrintTeamGoal` for outputting the bot's current team objective
- Declares the `notleader` array used to track which clients have been flagged as non-leaders across the bot subsystem

## External Dependencies
- **`bot_state_t`** — defined in `ai_main.h` or `g_local.h`; the central bot runtime state structure.
- **`MAX_CLIENTS`** — defined in `q_shared.h`; engine-wide client count limit.
- Implementation lives in `code/game/ai_cmd.c`.

# code/game/ai_dmnet.c
## File Purpose
Implements the bot AI finite-state machine (FSM) node system for deathmatch and team-game modes. Each `AINode_*` function is a discrete AI state (seek, battle, respawn, etc.) executed per-frame, and each `AIEnter_*` function transitions the bot into a new state. Also manages long-term goal (LTG) selection and navigation for all supported game types.

## Core Responsibilities
- Defines and drives the bot FSM: Intermission, Observer, Stand, Respawn, Seek_LTG, Seek_NBG, Seek_ActivateEntity, Battle_Fight, Battle_Chase, Battle_Retreat, Battle_NBG
- Selects and tracks long-term goals (LTG) based on game type (DM, CTF, 1FCTF, Obelisk, Harvester) and team strategy (defend, patrol, camp, escort, kill)
- Selects nearby goals (NBG) as short interruptions during LTG navigation
- Handles water/air survival logic via `BotGoForAir` / `BotGetAirGoal`
- Tracks node switches for AI debugging via `BotRecordNodeSwitch` / `BotDumpNodeSwitches`
- Detects and deactivates path obstacles (proximity mines, kamikaze bodies) via `BotClearPath`
- Selects a usable weapon for activation tasks via `BotSelectActivateWeapon`

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_team.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`, `ui/menudef.h`
- **Defined elsewhere:**
  - `bot_state_t`, `BotResetState`, `BotChat_*`, `BotFindEnemy`, `BotWantsToRetreat/Chase`, `BotAIPredictObstacles`, `BotAIBlocked`, `BotSetupForMovement`, `BotAttackMove`, `BotAimAtEnemy`, `BotCheckAttack`, `BotChooseWeapon`, `BotUpdateBattleInventory`, `BotBattleUseItems`, `BotMapScripts`, `BotTeamGoals`, `BotWantsToCamp`, `BotRoamGoal`, `BotAlternateRoute`, `BotGoHarvest` — all in companion `ai_*.c` files
  - `gametype`, `ctf_redflag`, `ctf_blueflag`, `ctf_neutralflag`, `redobelisk`, `blueobelisk`, `neutralobelisk` — game-mode globals from `ai_team.c` / `ai_dmq3.c`
  - All `trap_*` functions — game-module syscall stubs in `g_syscalls.c`

# code/game/ai_dmnet.h
## File Purpose
Public interface header for the Quake III Arena deathmatch bot AI state machine. It declares the state-enter functions (`AIEnter_*`) and state-node functions (`AINode_*`) that implement the bot's high-level behavioral FSM, along with diagnostic utilities.

## Core Responsibilities
- Declare the FSM state-entry transition functions (`AIEnter_*`) called when a bot switches states
- Declare the FSM state-node execution functions (`AINode_*`) called each frame to run the current state's logic
- Export node-switch diagnostic helpers for debugging bot behavior
- Define the `MAX_NODESWITCHES` guard constant to cap FSM transition history

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` (game-side bot state structure)
- `ai_dmnet.c` — provides all implementations declared here
- Consumers: `ai_main.c`, `ai_dmq3.c`, team AI files

# code/game/ai_dmq3.c
## File Purpose
Core Quake III Arena bot deathmatch AI implementation. It handles per-frame bot decision-making, enemy detection, combat behavior, team goal selection across all multiplayer gametypes (DM, CTF, 1FCTF, Obelisk, Harvester), obstacle avoidance, and game event processing.

## Core Responsibilities
- Per-frame bot AI tick (`BotDeathmatchAI`) that drives the AI node state machine
- Team goal selection: flag capture, base defense, escort, rush-base, harvest, obelisk attack
- Enemy detection and visibility testing with fog/water attenuation
- Aim prediction (linear and physics-based) and attack decision gating
- Inventory and battle inventory updates from `playerState_t`
- Dynamic obstacle detection and BSP entity activation (buttons, doors, trigger_multiples)
- Game event processing (obituaries, flag status changes, grenade/proxmine avoidance)
- Waypoint pool management and alternative route goal setup

## External Dependencies
- `g_local.h` — `gentity_t`, `level`, `g_entities[]`, `G_ModelIndex`, game trap functions
- `botlib.h` / `be_aas.h` / `be_ea.h` / `be_ai_*.h` — botlib AAS, EA, and AI API
- `ai_main.h` — `bot_state_t`, `BotAI_Print`, `BotAI_Trace`, `BotAI_GetEntityState`, `FloatTime`, `NumBots`, `AINode_*` enums, `AIEnter_*` functions
- `ai_dmnet.h` — `BotTeamAI`, `BotTeamLeader`, `AIEnter_Seek_LTG`, `AIEnter_Stand`, `AIEnter_Seek_ActivateEntity`, `BotValidChatPosition`, node switch utilities
- `ai_chat.h` / `ai_cmd.h` / `ai_team.h` — `BotVoiceChat`, `BotChat_EnterGame`, `BotMatchMessage`, `BotChatTime`, `BotSameTeam` (re-exported here), `BotSetTeamStatus`
- `chars.h`, `inv.h`, `syn.h`, `match.h` — characteristic indices, inventory indices, synonym/match contexts
- `ui/menudef.h` — voice chat string constants
- **Defined elsewhere (called but not defined here):** `BotEntityInfo`, `BotAI_GetClientState`, `BotAI_GetSnapshotEntity`, `BotVisibleTeamMatesAndEnemies` (partially defined here but also referenced by external callers), `trap_AAS_*`, `trap_EA_*`, `trap_Bot*`, `trap_Characteristic_*`

# code/game/ai_dmq3.h
## File Purpose
Public interface header for Quake III Arena's deathmatch bot AI subsystem. It declares all functions and extern symbols used by `ai_dmq3.c` and consumed by the broader game-side bot framework (primarily `ai_main.c`).

## Core Responsibilities
- Declare the bot deathmatch AI lifecycle functions (setup, shutdown, per-frame think)
- Expose combat decision helpers (enemy detection, weapon selection, aggression, retreat logic)
- Declare movement, inventory, and situational-awareness utilities
- Expose CTF and (conditionally) Mission Pack game-mode goal-setting routines
- Declare waypoint management functions
- Export global game-state variables and cvars used across bot AI files

## External Dependencies
- `bot_state_t`, `bot_waypoint_t`, `bot_goal_t`, `bot_moveresult_t`, `bot_activategoal_t` — defined in `ai_main.h` / `g_local.h`
- `aas_entityinfo_t` — defined in `be_aas.h` / botlib headers
- `vmCvar_t` — defined in `q_shared.h` / `qcommon.h`
- `vec3_t`, `qboolean` — defined in `q_shared.h`
- CTF flag constants (`CTF_FLAG_NONE/RED/BLUE`) and skin macros defined in this file; consumed by `ai_dmq3.c` and CTF-aware callers

# code/game/ai_main.c
## File Purpose
Central bot AI management module for Quake III Arena. It handles bot lifecycle (setup, shutdown, per-frame thinking), bridges the game module and the botlib, and converts bot AI decisions into usercmd_t inputs submitted to the server.

## Core Responsibilities
- Initialize and shut down the bot library and per-bot state (`BotAISetup`, `BotAIShutdown`, `BotAISetupClient`, `BotAIShutdownClient`)
- Drive per-frame bot thinking via `BotAIStartFrame`, dispatching `BotAI` for each active bot
- Translate bot inputs (`bot_input_t`) into network-compatible `usercmd_t` commands
- Manage view-angle interpolation and smoothing for bots
- Feed entity state updates into the botlib each frame
- Implement bot interbreeding (genetic algorithm) for fuzzy-logic goal evolution
- Persist and restore per-bot session data across map restarts

## External Dependencies

- `g_local.h` / `g_public.h` — game entity types, trap functions, game globals (`g_entities`, `level`, `maxclients`, `gametype`)
- `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h` — botlib API: AAS, elementary actions, chat/goal/move/weapon AI
- `ai_dmq3.h` / `ai_dmnet.h` / `ai_chat.h` / `ai_cmd.h` / `ai_vcmd.h` — higher-level deathmatch AI (`BotDeathmatchAI`, `BotSetupDeathmatchAI`, `BotChat_ExitGame`, etc.)
- `chars.h`, `inv.h`, `syn.h` — bot character, inventory, and synonym constants
- `trap_*` functions — VM syscall interface to the engine (AAS, EA, BotLib, Cvar, Trace, etc.), defined elsewhere in the engine/game syscall layer
- `ExitLevel` — declared extern, defined in `g_main.c`

# code/game/ai_main.h
## File Purpose
Central header for Quake III Arena's in-game bot AI system. Defines the monolithic `bot_state_t` structure that tracks all per-bot runtime state, along with constants for bot behavior flags, long-term goal types, team/CTF strategies, and shared AI utility function declarations.

## Core Responsibilities
- Define all bot behavioral flag constants (`BFL_*`) and long-term goal type constants (`LTG_*`)
- Define goal dedication timeouts for team and CTF scenarios
- Declare `bot_state_t`, the master per-bot state record spanning movement, goals, combat, team, and CTF data
- Declare `bot_waypoint_t` for checkpoint/patrol point linked lists
- Declare `bot_activategoal_t` for a stack of interactive object activation goals
- Expose the `FloatTime()` macro and utility function declarations used across AI subsystems

## External Dependencies
- `bg_public.h` / game headers: `playerState_t`, `usercmd_t`, `entityState_t`, `vec3_t`
- `botlib.h` / `be_aas.h`: `bot_goal_t`, `bot_settings_t`, `aas_entityinfo_t`, `bsp_trace_t`
- `be_ai_move.h`: move state handle type (used in `ms` field)
- Trap functions (`trap_Trace`, `trap_Printf`, etc.) — defined in `g_syscalls.c`, called from game VM

# code/game/ai_team.c
## File Purpose
Implements the bot team AI leadership system for Quake III Arena, responsible for issuing tactical orders to teammates based on game mode (Team DM, CTF, 1FCTF, Obelisk, Harvester). A single bot acts as team leader and periodically distributes role assignments (defend/attack/escort) to teammates sorted by proximity to the base.

## Core Responsibilities
- Validate and elect a team leader (human or bot)
- Count teammates and sort them by AAS travel time to the team's home base/obelisk
- Re-sort teammates by stored task preferences (defender/attacker/roamer)
- Issue context-sensitive orders per game mode and flag/objective status
- Deliver orders via team chat messages and/or voice chat commands (MISSIONPACK)
- Periodically re-evaluate strategy (randomly toggle aggressive/passive CTF strategy)

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h`, `ai_vcmd.h`, `match.h`, `../../ui/menudef.h`
- **Defined elsewhere:** `ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk` (goal structs from `ai_dmq3.c`/`ai_main.c`); `gametype`, `notleader[]`, `g_entities[]`; `BotSameTeam`, `BotTeam`, `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotPointAreaNum`, `ClientName`, `ClientFromName`, `FloatTime`, `BotSetLastOrderedTask`, `BotVoiceChat_Defend`; all `trap_*` syscalls

# code/game/ai_team.h
## File Purpose
Public interface header for Quake III Arena's bot team AI module. Declares the entry points and utility functions used by other game modules to drive team-based bot behavior and voice communication.

## Core Responsibilities
- Expose the main team AI tick function (`BotTeamAI`) for per-frame bot updates
- Provide teammate task preference get/set API for coordinating team roles
- Declare voice chat dispatch functions for bot-to-client voice communication

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` / `g_local.h` (defined elsewhere)
- Implementation bodies reside in `code/game/ai_team.c`

# code/game/ai_vcmd.c
## File Purpose
Handles bot AI responses to voice chat commands issued by human teammates. It maps incoming voice chat strings to specific bot behavioral state changes, enabling human players to direct bot teammates using in-game voice commands.

## Core Responsibilities
- Parse and dispatch incoming voice chat commands to handler functions
- Assign new long-term goal (LTG) types to bots in response to orders (get flag, defend, camp, follow, etc.)
- Validate gametype and team membership before acting on commands
- Manage bot leadership state (`teamleader`, `notleader`)
- Record task preferences for teammates (attacker vs. defender)
- Reset bot goal state when ordered to patrol (dismiss)
- Send acknowledgment chat/voice responses back to the commanding client

## External Dependencies
- `g_local.h` — `bot_state_t`, game globals (`gametype`, `ctf_redflag`, etc.), trap functions
- `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h` — helper functions (`BotSetTeamStatus`, `BotRememberLastOrderedTask`, `BotTeamFlagCarrier`, `BotGetAlternateRouteGoal`, `BotSameTeam`, `BotTeam`, etc.)
- `be_aas.h` — `aas_entityinfo_t`, `BotPointAreaNum`, `BotEntityInfo`
- `be_ai_chat.h`, `be_ea.h` — chat/action emission
- `ui/menudef.h` — `VOICECHAT_*` string constants
- `match.h`, `inv.h`, `syn.h`, `chars.h` — bot AI data constants
- **Defined elsewhere:** `notleader[]` array, goal globals (`ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk`), all `LTG_*` / `TEAM_*_TIME` constants, `FloatTime`, `random`

# code/game/ai_vcmd.h
## File Purpose
Public interface header for bot voice chat command handling in Quake III Arena's game-side AI system. Declares functions used to process and respond to voice chat events as part of bot behavioral logic.

## Core Responsibilities
- Expose the bot voice chat command dispatcher (`BotVoiceChatCommand`)
- Expose the "defend" voice chat response handler (`BotVoiceChat_Defend`)
- Serve as the include boundary between `ai_vcmd.c` and other game AI modules

## External Dependencies
- `bot_state_t` — defined elsewhere, likely `ai_main.h`
- Implementation body: `ai_vcmd.c` (noted in `$Archive` comment)
- No standard library includes in this header

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

## External Dependencies
- `qboolean`, `vec3_t` — defined in `q_shared.h` (engine shared types)
- `cplane_t` — referenced in the commented-out `bsp_trace_t` block; defined in `q_shared.h`
- `botlib.h` — noted inline as the canonical home for `bsp_trace_t` / `bsp_surface_t` (excluded via comment guard)
- `MAX_STRINGFIELD` — guarded define, may also be provided by botlib headers

# code/game/be_ai_char.h
## File Purpose
Public API header for the bot character system, exposing functions to load, query, and free bot personality/skill profiles. It defines the interface through which game code retrieves typed characteristic values (float, integer, string) from a named character file.

## Core Responsibilities
- Declare the bot character load/free lifecycle functions
- Expose typed accessors for individual bot characteristics by index
- Provide bounded variants of numeric accessors to clamp values within caller-specified ranges
- Declare a global shutdown function to release all cached character data

## External Dependencies
- No includes in this header.
- All function bodies defined in `code/botlib/be_ai_char.c` (defined elsewhere).
- Consumed via the botlib interface layer (`be_interface.c`) or directly by game bot code.

# code/game/be_ai_chat.h
## File Purpose
Declares the public interface for the bot chat AI subsystem, defining data structures and function prototypes used to manage bot console message queues, pattern-based chat matching, and chat message generation/delivery.

## Core Responsibilities
- Define constants for message size limits, gender flags, and chat target types
- Declare the console message linked-list node structure for per-bot message queues
- Declare match variable and match result structures for template-based message parsing
- Expose lifecycle functions for the chat AI subsystem (setup/shutdown, alloc/free state)
- Expose functions for queuing, retrieving, and removing console messages
- Expose functions for selecting, composing, and sending chat replies
- Expose utility functions for string matching, synonym replacement, and whitespace normalization

## External Dependencies
- No includes visible in this header; implementation resides in `botlib/be_ai_chat.c`.
- `MAX_MESSAGE_SIZE`, `MAX_MATCHVARIABLES`, gender/target constants are self-contained in this file.
- All function bodies are **defined elsewhere** (botlib shared library, linked via `botlib_export_t` function table).

# code/game/be_ai_gen.h
## File Purpose
Public header exposing the genetic selection interface used by the bot AI system. It declares a single utility function for selecting parent and child candidates based on a ranked fitness array, supporting evolutionary/genetic algorithm techniques in bot decision-making.

## Core Responsibilities
- Declare the `GeneticParentsAndChildSelection` interface for use by bot AI modules
- Expose genetic selection logic as a callable contract across translation units

## External Dependencies
- No includes in this header.
- Implementation defined elsewhere: `code/botlib/be_ai_gen.c`
- Consumed by: `code/game/` bot AI modules and potentially `code/botlib/` internals

# code/game/be_ai_goal.h
## File Purpose
Public interface header for the bot goal AI subsystem in Quake III Arena's botlib. It defines the `bot_goal_t` structure and declares all functions used to manage bot goals, goal stacks, item weights, and fuzzy logic for goal selection.

## Core Responsibilities
- Define the `bot_goal_t` structure representing a navigable destination
- Declare goal state lifecycle management (alloc, reset, free)
- Declare goal stack push/pop/query operations
- Declare long-term goal (LTG) and nearby goal (NBG) item selection
- Declare item weight loading and fuzzy logic mutation/interbreeding
- Declare level item initialization and dynamic entity item updates
- Declare avoid-goal tracking and timing

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `MAX_AVOIDGOALS`, `MAX_GOALSTACK`, `GFL_*` flags — defined in this file
- All function bodies — defined in `code/botlib/be_ai_goal.c`
- AAS travel flag constants (`travelflags`) — defined in `be_aas.h` / `aasfile.h`

# code/game/be_ai_move.h
## File Purpose
Public header defining the movement AI interface for Quake III's bot library. It declares movement type flags, move state flags, result flags, key data structures, and the full function API used by game code to drive bot locomotion.

## Core Responsibilities
- Define bitmask constants for movement types (walk, crouch, jump, grapple, rocket jump)
- Define bitmask constants for movement state flags (on-ground, swimming, teleported, etc.)
- Define bitmask constants for movement result flags (view override, blocked, obstacle, elevator)
- Declare `bot_initmove_t` for seeding a move state from player/entity state
- Declare `bot_moveresult_t` for communicating locomotion outcomes back to callers
- Declare `bot_avoidspot_t` for spatial hazard avoidance regions
- Expose the full movement AI lifecycle API (alloc/init/move/free)

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `bot_goal_t` — defined in `be_ai_goal.h`
- `bot_initmove_t.or_moveflags` values (`MFL_ONGROUND`, etc.) sourced from engine `playerState_t` by the caller
- Implementation: `code/botlib/be_ai_move.c`

# code/game/be_ai_weap.h
## File Purpose
Public header defining data structures and function prototypes for the bot weapon AI subsystem. It describes projectile and weapon properties used by the botlib to reason about weapon selection and ballistics.

## Core Responsibilities
- Define flags for projectile behavior (window damage, return-to-owner)
- Define flags for weapon firing behavior (key-up fire release)
- Define damage type bitmasks (impact, radial, visible)
- Declare `projectileinfo_t` and `weaponinfo_t` structs used throughout the bot weapon system
- Expose the weapon AI lifecycle API (setup, shutdown, alloc, free, reset)
- Expose weapon selection and information query functions

## External Dependencies
- `MAX_STRINGFIELD` — defined in botlib shared headers (e.g., `botlib.h` or `be_aas.h`)
- `vec3_t` — defined in `q_shared.h`
- All function bodies defined in `code/botlib/be_ai_weap.c`

# code/game/be_ea.h
## File Purpose
Declares the "Elementary Actions" (EA) API for the Quake III bot library. It provides the bot system's lowest-level abstraction over client input, translating high-level bot decisions into discrete client commands and movement/view inputs that are eventually forwarded to the server.

## Core Responsibilities
- Declare client-command EA functions (chat, arbitrary commands, discrete button actions)
- Declare movement EA functions (crouch, walk, strafe, jump, directional move)
- Declare view/weapon EA functions (aim direction, weapon selection)
- Declare input aggregation and dispatch functions (end-of-frame flush, input readback, reset)
- Declare module lifecycle entry points (setup/shutdown)

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `bot_input_t` — defined in `botlib.h` / `be_aas_def.h`
- Implementation: `botlib/be_ea.c`
- Consumed by: `game/ai_move.c`, `game/ai_dmq3.c`, and other game-side AI modules via the botlib interface

# code/game/bg_lib.c
## File Purpose
A self-contained replacement for the standard C library, compiled exclusively for use in Quake III's virtual machine (Q3_VM) target. It provides `qsort`, string functions, math functions, printf-family functions, and numeric parsing so that VM-compiled game modules (game, cgame, ui) do not depend on the host platform's libc.

## Core Responsibilities
- Provide a portable `qsort` (Bentley-McIlroy) usable in both VM and native builds
- Supply string functions (`strlen`, `strcpy`, `strcat`, `strcmp`, `strchr`, `strstr`) for VM builds
- Supply character-classification helpers (`tolower`, `toupper`) for VM builds
- Provide table-driven trigonometry (`sin`, `cos`, `acos`, `atan2`) and `tan` for VM builds
- Implement numeric conversion (`atoi`, `atof`, `_atoi`, `_atof`) with pointer-advance variants
- Implement a minimal `vsprintf`/`sscanf` for formatted I/O inside the VM
- Provide `memmove`, `rand`/`srand`, `abs`, `fabs`

## External Dependencies
- **Includes:** `q_shared.h` (provides `qtrue`, `M_PI`, `size_t`, `va_list`, and the `Q3_VM` macro)
- **Defined elsewhere:** `cmp_t` is conditionally typedef'd here only when `Q3_VM` is not defined; under `Q3_VM` it is assumed provided by `bg_lib.h` (included via `q_shared.h → bg_lib.h`)
- **No heap allocation:** All functions operate on caller-supplied buffers or static/stack storage

# code/game/bg_lib.h
## File Purpose
A self-contained replacement header for standard C library declarations, intended exclusively for use when compiling game code targeting the Quake III virtual machine (QVM). It is explicitly not included in native host builds.

## Core Responsibilities
- Provides `size_t` and `va_list` type definitions for the VM environment
- Declares integer limit macros (`INT_MAX`, `CHAR_BIT`, etc.) normally found in `<limits.h>`
- Declares variadic argument macros (`va_start`, `va_arg`, `va_end`) normally from `<stdarg.h>`
- Declares string manipulation function prototypes replacing `<string.h>`
- Declares memory operation prototypes replacing `<string.h>`/`<memory.h>`
- Declares math function prototypes replacing `<math.h>`
- Declares misc stdlib prototypes (`qsort`, `rand`, `atoi`, `atof`, etc.) replacing `<stdlib.h>`

## External Dependencies
- No includes — this file is itself the bottom of the dependency chain for VM builds.
- All declared symbols are **defined in** `code/game/bg_lib.c` (not inferable from this file alone, but implied by the file comment).
- `va_start`/`va_arg`/`va_end` macros assume a simple cdecl-style stack layout matching the QVM's int-aligned argument passing; they are **not** portable to x86-64 or other ABIs and must never be used in native builds.

# code/game/bg_local.h
## File Purpose
Internal header for the "bg" (both-game) player movement subsystem, shared between the game server and client-side prediction code. It declares the private `pml_t` locals struct, physics tuning constants, and exposes internal pmove helper function signatures that are used across the `bg_pmove.c` and `bg_slidemove.c` translation units.

## Core Responsibilities
- Define movement physics constants (slope limits, step height, jump velocity, timers)
- Declare `pml_t`, the per-frame local movement state that is zeroed before every `Pmove` call
- Expose `pm` and `pml` as extern globals shared across bg source files
- Declare extern movement parameter floats (speed, acceleration, friction tuning values)
- Expose the internal utility function prototypes used only within the bg subsystem

## External Dependencies
- **`q_shared.h` / `bg_public.h`** — `vec3_t`, `trace_t`, `qboolean`, `pmove_t` types (defined elsewhere)
- `pmove_t` — defined in `bg_public.h`
- `vec3_t`, `trace_t` — defined in `q_shared.h`
- All `extern` variables are **defined** in `bg_pmove.c`

# code/game/bg_misc.c
## File Purpose
Defines the master item registry (`bg_itemlist`) for all pickups in Quake III Arena and provides stateless utility functions shared between the server game and client game modules for item lookup, trajectory evaluation, player state conversion, and event management.

## Core Responsibilities
- Declares and initializes the global `bg_itemlist[]` array containing every item definition (weapons, ammo, armor, health, powerups, holdables, team items)
- Provides item lookup functions by powerup tag, holdable tag, weapon tag, and pickup name
- Implements trajectory position and velocity evaluation for all `trType_t` variants
- Determines whether a player can pick up a given item (`BG_CanItemBeGrabbed`) with full gametype/team/MISSIONPACK awareness
- Tests spatial proximity between a player and an item entity
- Manages the predictable event ring-buffer in `playerState_t`
- Converts `playerState_t` → `entityState_t` (both interpolated and extrapolated variants)
- Handles jump-pad velocity application and event generation

## External Dependencies
- `q_shared.h` — core math macros (`VectorCopy`, `VectorMA`, `VectorScale`, `VectorClear`, `SnapVector`), type definitions (`vec3_t`, `playerState_t`, `entityState_t`, `trajectory_t`, `qboolean`), `Com_Error`, `Com_Printf`, `Q_stricmp`, `AngleNormalize180`, `vectoangles`
- `bg_public.h` — `gitem_t`, `itemType_t`, `powerup_t`, `holdable_t`, `weapon_t`, `entity_event_t`, `gametype_t`, `DEFAULT_GRAVITY`, `GIB_HEALTH`, `STAT_*`, `PERS_*`, `PW_*`, `HI_*`, `WP_*`, `EV_*`, `ET_*`, `TR_*`
- `trap_Cvar_VariableStringBuffer` — declared (not defined) in this file; resolved at link time against the VM trap table (cgame or game module)
- `sin`, `cos`, `fabs` — C math library (or VM substitutes via `bg_lib`)

# code/game/bg_pmove.c
## File Purpose
Implements the core player movement (pmove) system for Quake III Arena, shared between the server and client game modules. Takes a `pmove_t` (containing a `playerState_t` and `usercmd_t`) as input and produces an updated `playerState_t` as output. Designed for deterministic client-side prediction.

## Core Responsibilities
- Simulates all player movement modes: walking, air, water, fly, noclip, grapple, dead, spectator
- Applies friction and acceleration per-medium (ground, water, flight, spectator)
- Detects and handles ground contact, slope clamping, and the "all solid" edge case
- Manages jump, crouch/duck, water level, and water jump logic
- Drives weapon state transitions (raising, dropping, firing, ammo consumption)
- Drives legs and torso animation state machines via toggle-bit animation indices
- Generates predictable player events (footsteps, splashes, fall damage, weapon fire, etc.)
- Chops long frames into sub-steps via `Pmove` to prevent framerate-dependent behavior

## External Dependencies

- **Includes:** `q_shared.h`, `bg_public.h`, `bg_local.h`
- **Defined elsewhere:**
  - `PM_SlideMove`, `PM_StepSlideMove` — defined in `bg_slidemove.c`
  - `BG_AddPredictableEventToPlayerstate` — defined in `bg_misc.c`
  - `trap_SnapVector` — syscall stub; platform-specific (snaps float vector components to integers)
  - `AngleVectors`, `VectorNormalize`, `DotProduct`, etc. — `q_shared.c` / `q_math.c`
  - `bg_itemlist` — defined in `bg_misc.c`
  - `Com_Printf` — engine/qcommon

# code/game/bg_public.h
## File Purpose
Shared header defining all game-logic constants, enumerations, and data structures used by both the server-side game module (`game`) and the client-side game module (`cgame`). It establishes the contract between those two VMs and the engine for entity state, player state, items, movement, and events.

## Core Responsibilities
- Define config-string indices (`CS_*`) for server-to-client communication
- Declare all game enumerations: game types, powerups, weapons, holdables, entity types, entity events, animations, means of death
- Define the `pmove_t` context struct and declare the `Pmove` / `PM_UpdateViewAngles` entry points
- Define `player_state` index enumerations (`statIndex_t`, `persEnum_t`)
- Declare the item system (`gitem_t`, `bg_itemlist`, `BG_Find*` helpers)
- Declare shared BG utility functions for trajectory evaluation, event injection, and state conversion
- Define Kamikaze effect timing and sizing constants

## External Dependencies
- `q_shared.h` — `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `vec3_t`, `trace_t`, `qboolean`, `CONTENTS_*`, `MAX_*` constants, `CS_SERVERINFO`/`CS_SYSTEMINFO`
- `MISSIONPACK` preprocessor define — gates additional weapons (`WP_NAILGUN`, `WP_PROX_LAUNCHER`, `WP_CHAINGUN`), powerups, means of death, and entity flags for the Team Arena expansion
- All `BG_*` function bodies defined in `bg_misc.c`, `bg_pmove.c`, `bg_slidemove.c`, `bg_lib.c`

# code/game/bg_slidemove.c
## File Purpose
Implements the sliding collision-response movement for the Quake III pmove system. It resolves player velocity against world geometry by iteratively tracing and clipping velocity along collision planes, and handles automatic step-up over ledges.

## Core Responsibilities
- Trace player movement each frame and clip velocity against hit planes
- Handle up to `MAX_CLIP_PLANES` (5) simultaneous collision planes per move iteration
- Apply gravity interpolation during slide moves
- Detect and resolve two-plane crease collisions via cross-product projection
- Stop the player dead on triple-plane interactions
- Step up over geometry up to `STEPSIZE` (18 units) high via `PM_StepSlideMove`
- Fire step-height events (`EV_STEP_4/8/12/16`) for audio/animation feedback

## External Dependencies
- `q_shared.h` — `vec3_t`, `trace_t`, `qboolean`, vector math macros (`DotProduct`, `VectorMA`, `CrossProduct`, etc.)
- `bg_public.h` — `pmove_t`, `playerState_t`, `EV_STEP_*` event enums, `MAXTOUCH`
- `bg_local.h` — `pml_t`, `STEPSIZE`, `OVERCLIP`, `JUMP_VELOCITY`; extern declarations for `pm`, `pml`, `c_pmove`; declarations of `PM_ClipVelocity`, `PM_AddTouchEnt`, `PM_AddEvent`
- **Defined elsewhere:** `PM_ClipVelocity` (bg_pmove.c), `PM_AddTouchEnt` (bg_pmove.c), `PM_AddEvent` (bg_pmove.c), `pm->trace` callback (set by caller in game/cgame), `Com_Printf` (engine)

# code/game/botlib.h
## File Purpose
Defines the public API boundary between the Quake III game module and the bot AI library (botlib). It declares all function pointer tables (vtables) used to import engine services into botlib and export bot subsystem capabilities back to the game.

## Core Responsibilities
- Define the versioned `botlib_export_t` / `botlib_import_t` interface structs
- Declare input/state types (`bot_input_t`, `bot_entitystate_t`, `bsp_trace_t`) shared across the boundary
- Group bot subsystem exports into nested vtable structs: `aas_export_t`, `ea_export_t`, `ai_export_t`
- Define action flag bitmasks used to encode bot commands
- Define error codes (`BLERR_*`) and print type constants for botlib diagnostics
- Document all configurable library variables and their defaults in a reference comment block

## External Dependencies
- `vec3_t`, `cplane_t`, `qboolean` — defined in `q_shared.h`
- `fileHandle_t`, `fsMode_t` — defined in `q_shared.h` / `qcommon.h`
- `pc_token_t` — defined in the botlib script/precompiler headers (`l_precomp.h`)
- Forward-declared structs (`aas_clientmove_s`, `bot_goal_s`, etc.) — defined in respective `be_aas_*.h` / `be_ai_*.h` headers
- `QDECL` calling-convention macro — defined in `q_shared.h`

# code/game/chars.h
## File Purpose
Defines integer constants (characteristic indices) used to index into a bot's personality/behavior data structure. Each constant maps a named behavioral trait to a slot number understood by the bot AI and botlib systems.

## Core Responsibilities
- Enumerate all bot characteristic slot indices (0–48)
- Categorize traits into logical groups: identity, combat, chat, movement, and goal-seeking
- Provide a shared vocabulary between the game module and botlib for reading/writing bot personality values

## External Dependencies
- No `#include` directives; this header is self-contained.
- **Defined elsewhere / consumers:**
  - `botlib/be_ai_char.c` — reads/writes characteristic values using these indices
  - `game/ai_main.c`, `game/ai_dmq3.c`, etc. — pass these constants to botlib API calls such as `trap_Characteristic_Float` / `trap_Characteristic_String`
  - `botlib/botlib.h` — declares the `BotCharacteristic_*` API that accepts these index values


# code/game/g_active.c
## File Purpose
Implements per-client per-frame logic for the server-side game module, covering player movement, environmental effects, damage feedback, event dispatch, and end-of-frame state synchronization. It is the central "think" driver for all connected clients each server frame.

## Core Responsibilities
- Run `Pmove` physics simulation for each client and propagate results back to entity state
- Apply world environmental damage (drowning, lava, slime) each frame
- Aggregate and encode damage feedback into `playerState_t` for pain blends/kicks
- Dispatch and process server-authoritative client events (falling, weapon fire, item use, teleport)
- Handle spectator movement and chase-cam follow logic
- Enforce inactivity kick timer and respawn conditions
- Execute once-per-second timer actions (health regen, armor decay, ammo regen via MISSIONPACK)
- Synchronize `playerState_t` → `entityState_t` and send predictable events to other clients

## External Dependencies
- `g_local.h` (pulls `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `Pmove` (bg_pmove.c), `G_Damage`, `G_AddEvent`, `G_Sound`, `G_TempEntity`, `G_SoundIndex` (g_utils/g_combat), `BG_PlayerStateToEntityState`, `BG_PlayerTouchesItem` (bg_misc.c), `FireWeapon`, `CheckGauntletAttack`, `Weapon_HookFree` (g_weapon.c), `TeleportPlayer`, `SelectSpawnPoint`, `respawn` (g_client/g_misc), `Drop_Item` (g_items.c), `BotTestAAS` (ai_main.c), all `trap_*` syscalls (g_syscalls.c)

# code/game/g_arenas.c
## File Purpose
Manages the post-game intermission sequence for Quake III Arena's single-player and tournament modes, including spawning player model replicas on victory podiums and assembling the `postgame` server command that drives the end-of-match scoreboard/stats UI.

## Core Responsibilities
- Collect and format end-of-match statistics into a `postgame` console command sent to all clients
- Spawn a physical podium entity in the intermission zone
- Spawn static player body replicas on the podium for the top 3 finishers
- Continuously reorient the podium and its occupants toward the intermission camera via a think function
- Drive the winner's celebration (gesture) animation with a timed start/stop
- Provide a server command (`Svcmd_AbortPodium_f`) to cancel the podium celebration in single-player

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `level` (`level_locals_t`), `g_entities[]`, `g_gametype`; trap functions (`trap_SendConsoleCommand`, `trap_LinkEntity`, `trap_Cvar_VariableIntegerValue`); math utilities (`AngleVectors`, `VectorMA`, `vectoangles`, `vectoyaw`); entity helpers (`G_Spawn`, `G_SetOrigin`, `G_ModelIndex`, `G_AddEvent`, `G_Printf`); `CalculateRanks`; `SP_PODIUM_MODEL` (defined in `g_local.h`).

# code/game/g_bot.c
## File Purpose
Manages bot lifecycle within the game module: loading bot/arena definitions from data files, adding/removing bots dynamically, maintaining a deferred spawn queue, and enforcing minimum player counts per game type.

## Core Responsibilities
- Parse and cache bot info records from `scripts/bots.txt` and `.bot` files
- Parse and cache arena info records from `scripts/arenas.txt` and `.arena` files
- Allocate client slots and build userinfo strings when adding a bot (`G_AddBot`)
- Maintain a fixed-depth spawn queue (`botSpawnQueue`) to stagger bot `ClientBegin` calls
- Enforce `bot_minplayers` cvar by adding/removing random bots each 10-second interval
- Expose server commands `addbot` and `botlist` via `Svcmd_AddBot_f` / `Svcmd_BotList_f`
- Initialize single-player mode bots with correct fraglimit/timelimit from arena info

## External Dependencies
- `g_local.h` — all shared game types, trap declarations, `level`, `g_entities`, cvars
- `BotAISetupClient`, `BotAIShutdown` — defined in `ai_main.c` (botlib AI layer)
- `ClientConnect`, `ClientBegin`, `ClientDisconnect` — defined in `g_client.c`
- `G_Alloc` — defined in `g_mem.c`
- `PickTeam` — defined in `g_client.c`
- `podium1/2/3` — `extern gentity_t*` owned by `g_arenas.c`
- `COM_Parse`, `COM_ParseExt`, `Info_SetValueForKey`, `Info_ValueForKey`, `Q_strncpyz` — defined in `q_shared.c` / `bg_lib.c`
- All `trap_*` functions — syscall stubs resolved by the VM/engine boundary

# code/game/g_client.c
## File Purpose
Manages the full client lifecycle within the game module: connection, spawning, respawning, userinfo updates, body queue management, and disconnection. Handles spawn point selection logic and player state initialization at each spawn.

## Core Responsibilities
- Spawn point registration (`SP_info_player_*`) and selection (nearest, random, furthest, initial, spectator)
- Body queue management: pooling corpse entities, animating their sink/disappearance
- Client lifecycle callbacks: `ClientConnect`, `ClientBegin`, `ClientSpawn`, `ClientDisconnect`
- Userinfo parsing and configstring broadcasting (`ClientUserinfoChanged`)
- Player name sanitization (`ClientCleanName`)
- Team utility queries: `TeamCount`, `TeamLeader`, `PickTeam`
- View angle delta computation (`SetClientViewAngle`)

## External Dependencies

- **Includes:** `g_local.h` (which pulls `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `ClientThink`, `ClientEndFrame` — `g_active.c`
  - `SelectCTFSpawnPoint` — `g_team.c`
  - `BG_PlayerStateToEntityState` — `bg_misc.c`
  - `FindIntermissionPoint`, `MoveClientToIntermission` — `g_main.c` / `p_hud.c`
  - `TossClientItems`, `body_die` — `g_combat.c`
  - `G_BotConnect`, `BotAIShutdownClient` — `g_bot.c` / `ai_main.c`
  - `CalculateRanks`, `BroadcastTeamChange` — `g_main.c` / `g_cmds.c`
  - All `trap_*` functions — server syscall stubs in `g_syscalls.c`

# code/game/g_cmds.c
## File Purpose
Implements all client-side command handlers for the Quake III Arena game module. It serves as the primary dispatcher (`ClientCommand`) that maps incoming client command strings to their respective handler functions, covering chat, team management, voting, spectating, and cheat commands.

## Core Responsibilities
- Parse and dispatch client commands via `ClientCommand`
- Handle chat and voice communication (`say`, `say_team`, `tell`, voice variants)
- Manage team assignment and spectator follow modes
- Implement cheat commands (god, noclip, notarget, give)
- Implement voting system (callvote, vote, callteamvote, teamvote)
- Build and send scoreboard data to clients
- Handle taunt/voice chat logic including context-aware insult selection

## External Dependencies
- **Includes:** `g_local.h` (all game types, trap functions, globals), `../../ui/menudef.h` (VOICECHAT_* string constants)
- **Defined elsewhere:** `level` (`level_locals_t` global from `g_main.c`), `g_entities` array, all `trap_*` syscalls (resolved by the engine VM), `player_die`, `BeginIntermission`, `TeleportPlayer`, `CopyToBodyQue`, `ClientUserinfoChanged`, `ClientBegin`, `SetLeader`, `CheckTeamLeader`, `PickTeam`, `TeamCount`, `TeamLeader`, `OnSameTeam`, `Team_GetLocationMsg`, `BG_FindItem`, `G_Spawn`, `G_SpawnItem`, `FinishSpawningItem`, `Touch_Item`, `G_FreeEntity`, `G_LogPrintf`, `G_Printf`, `G_Error`

# code/game/g_combat.c
## File Purpose
Implements all server-side combat logic for Quake III Arena's game module, including damage application, knockback, scoring, death processing, item drops, and radius explosion damage. It serves as the central damage pipeline that all weapons and hazards funnel through.

## Core Responsibilities
- Apply damage to entities via `G_Damage`, handling armor absorption, knockback, godmode, team protection, and invulnerability
- Execute player death sequence via `player_die`, including obituary logging, scoring, animation, and flag/item handling
- Perform area-of-effect damage via `G_RadiusDamage` with line-of-sight gating
- Drop held weapons and powerups on player death via `TossClientItems`
- Manage score additions and visual score plums via `AddScore`/`ScorePlum`
- Handle gib deaths and body corpse state transitions via `GibEntity`/`body_die`
- Detect near-capture/near-score events for "holy shit" reward triggers

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `g_entities[]`, `level` — global game state (`g_main.c`)
  - `g_knockback`, `g_blood`, `g_friendlyFire`, `g_gametype`, `g_debugDamage`, `g_cubeTimeout` — cvars
  - `Team_FragBonuses`, `Team_ReturnFlag`, `Team_CheckHurtCarrier`, `OnSameTeam` — `g_team.c`
  - `Drop_Item`, `LaunchItem`, `BG_FindItemForWeapon`, `BG_FindItemForPowerup`, `BG_FindItem` — items/bg layer
  - `Weapon_HookFree`, `LogAccuracyHit` — `g_weapon.c`
  - `Cmd_Score_f` — `g_cmds.c`
  - `G_StartKamikaze` — `g_weapon.c` (MISSIONPACK)
  - `CheckObeliskAttack` — `g_team.c` (MISSIONPACK)
  - All `trap_*` functions — syscall interface to the server engine

# code/game/g_items.c
## File Purpose
Implements the server-side item system for Quake III Arena, handling pickup logic, item spawning, dropping, respawning, and per-frame physics simulation for all in-game collectibles (weapons, ammo, health, armor, powerups, holdables, and team items).

## Core Responsibilities
- Execute type-specific pickup logic and award appropriate effects to the picking client
- Manage item respawn timers and team-based item selection on respawn
- Spawn world items at map load, dropping them to floor via trace
- Launch and drop items dynamically (e.g., on player death)
- Simulate per-frame physics for airborne items (gravity, bounce, NODROP removal)
- Maintain the item registration/precache bitfield written to config strings
- Validate required team-game entities (flags, obelisks) at map start

## External Dependencies
- `g_local.h` → `q_shared.h`, `bg_public.h`, `g_public.h`
- **Defined elsewhere:**
  - `bg_itemlist`, `bg_numItems` — item table (`bg_misc.c`)
  - `BG_CanItemBeGrabbed`, `BG_FindItem`, `BG_FindItemForWeapon`, `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — shared game library
  - `Pickup_Team`, `Team_DroppedFlagThink`, `Team_CheckDroppedItem`, `Team_FreeEntity`, `Team_InitGame` — `g_team.c`
  - `G_Spawn`, `G_FreeEntity`, `G_TempEntity`, `G_UseTargets`, `G_SetOrigin`, `G_AddEvent`, `G_AddPredictableEvent`, `G_SoundIndex`, `G_RunThink` — `g_utils.c` / `g_main.c`
  - `trap_Trace`, `trap_LinkEntity`, `trap_PointContents`, `trap_SetConfigstring`, `trap_Cvar_VariableIntegerValue`, `trap_GetUserinfo` — engine syscall stubs
  - `g_weaponRespawn`, `g_weaponTeamRespawn`, `g_gametype` — cvars declared in `g_main.c`
  - `level` — global `level_locals_t` from `g_main.c`

# code/game/g_local.h
## File Purpose
Central private header for the Quake III Arena server-side game module (game DLL/VM). It defines all major game-side data structures, declares every cross-file function, enumerates game cvars, and lists the full `trap_*` syscall interface that bridges the game VM to the engine.

## Core Responsibilities
- Define `gentity_t` (the universal server-side entity) and `gclient_t` (per-client runtime state)
- Define `level_locals_t`, the singleton that holds all per-map game state
- Define `clientPersistant_t` and `clientSession_t` for data surviving respawns/levels
- Declare every public function exported between game `.c` files
- Declare all `vmCvar_t` globals used by the game module
- Declare all `trap_*` engine syscall wrappers (filesystem, collision, bot AI, etc.)
- Define entity flag bits (`FL_*`), damage flags (`DAMAGE_*`), and timing constants

## External Dependencies

- **Includes**: `q_shared.h` (base types, math, `entityState_t`, `playerState_t`), `bg_public.h` (shared game types: items, weapons, pmove, events), `g_public.h` (engine API enum, `sharedEntity_t`, `entityShared_t`), `g_team.h` (CTF/team function prototypes)
- **Defined elsewhere**:
  - `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t` — `q_shared.h`
  - `entityShared_t`, `gameImport_t` — `g_public.h`
  - `gitem_t`, `weapon_t`, `team_t`, `gametype_t` — `bg_public.h`
  - All `trap_*` function bodies — `g_syscalls.c` (VM syscall dispatch stubs)
  - `level`, `g_entities`, all `vmCvar_t` definitions — `g_main.c`

# code/game/g_main.c
## File Purpose
The central game module entry point for Quake III Arena's server-side game logic. It owns the VM dispatch table (`vmMain`), manages game initialization/shutdown, drives the per-frame update loop, and maintains all game-wide cvars and level state.

## Core Responsibilities
- Expose `vmMain` as the sole entry point from the engine into the game VM
- Register and update all server-side cvars via `gameCvarTable`
- Initialize and tear down the game world (`G_InitGame`, `G_ShutdownGame`)
- Drive the per-frame entity update loop (`G_RunFrame`)
- Manage tournament warmup, voting, team voting, and exit rules
- Compute and broadcast player/team score rankings
- Handle level intermission sequencing and map transitions

## External Dependencies
- `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`) — all shared types and trap declarations
- `trap_*` syscalls — defined in the engine, bridged through `g_syscalls.c`; cover FS, cvars, server commands, entity linking, AAS, bot lib, etc.
- `ClientConnect`, `ClientThink`, `ClientBegin`, `ClientDisconnect`, `ClientCommand`, `ClientUserinfoChanged`, `ClientEndFrame` — defined in `g_client.c` / `g_active.c`
- `BotAISetup`, `BotAIShutdown`, `BotAILoadMap`, `BotAIStartFrame`, `BotInterbreedEndMatch` — defined in `ai_main.c` / `g_bot.c`
- `G_SpawnEntitiesFromString`, `G_CheckTeamItems`, `UpdateTournamentInfo`, `SpawnModelsOnVictoryPads`, `CheckTeamStatus` — defined elsewhere in the game module

# code/game/g_mem.c
## File Purpose
Provides a simple bump-pointer memory allocator backed by a fixed 256 KB static pool for the game module. All allocations are permanent for the duration of a map session; there is no free operation.

## Core Responsibilities
- Allocate memory from a fixed-size static pool with 32-byte alignment
- Detect pool exhaustion and fatal-error on overflow
- Reset the pool at map/session start via `G_InitMemory`
- Expose current pool usage via a server console command

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `g_debugAlloc` — `vmCvar_t` extern declared in `g_local.h`, defined in `g_main.c`
  - `G_Printf` — defined in `g_main.c`; wraps `trap_Printf`
  - `G_Error` — defined in `g_main.c`; wraps `trap_Error` (non-returning)

# code/game/g_misc.c
## File Purpose
Implements miscellaneous map entity spawn functions and gameplay systems for the Quake III Arena game module, including teleportation logic, portal surfaces, positional markers, and trigger-based weapon shooters.

## Core Responsibilities
- Spawn and initialize editor-only or utility entities (`info_null`, `info_camp`, `light`, `func_group`)
- Implement the `TeleportPlayer` function used by trigger teleporters and portals
- Set up portal surface/camera pairs for in-world mirror/portal rendering
- Initialize trigger-based weapon shooter entities (`shooter_rocket`, `shooter_plasma`, `shooter_grenade`)
- Handle `#ifdef MISSIONPACK` portal item mechanics (drop source/destination pads)

## External Dependencies
- **`g_local.h`** — pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, all `gentity_t`/`gclient_t` definitions, and all `trap_*` syscall declarations.
- **Defined elsewhere:** `G_TempEntity`, `G_KillBox`, `G_PickTarget`, `G_SetMovedir`, `BG_PlayerStateToEntityState`, `SetClientViewAngle`, `fire_grenade`, `fire_rocket`, `fire_plasma`, `RegisterItem`, `BG_FindItemForWeapon`, `Drop_Item`, `BG_FindItemForPowerup`, `G_Damage`, `G_Find`, `G_Spawn`, `G_SetOrigin`, `G_FreeEntity`, `DirToByte`, `PerpendicularVector`, `CrossProduct`, `crandom`, `level` (global), all `trap_*` functions.

# code/game/g_missile.c
## File Purpose
Implements server-side missile entity creation, movement simulation, and impact handling for all projectile weapons in Quake III Arena. It spawns missile entities, advances them each frame via trajectory evaluation and collision tracing, and dispatches bounce, impact, or explosion logic on collision.

## Core Responsibilities
- Spawn typed missile entities (plasma, grenade, rocket, BFG, grapple, and MISSIONPACK: nail, prox mine)
- Advance missiles each server frame: evaluate trajectory, trace movement, detect collisions
- Handle missile impact: apply direct damage, splash damage, bounce, grapple attachment
- Manage MISSIONPACK proximity mine lifecycle: activation, trigger volumes, player-sticking, timed explosion
- Emit network events (hit/miss/bounce/explosion) for client-side effects
- Track accuracy hits on the owning client

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — `bg_pmove.c`/`bg_misc.c`
  - `G_Damage`, `G_RadiusDamage`, `CanDamage`, `G_InvulnerabilityEffect` — `g_combat.c`
  - `LogAccuracyHit`, `Weapon_HookFree`, `Weapon_HookThink`, `SnapVectorTowards` — `g_weapon.c`
  - `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `G_AddEvent`, `G_SoundIndex` — `g_utils.c`
  - `G_RunThink` — `g_main.c`
  - `trap_Trace`, `trap_LinkEntity` — engine syscall stubs (`g_syscalls.c`)
  - `level`, `g_entities`, `g_proxMineTimeout`, `g_gametype` — game module globals

# code/game/g_mover.c
## File Purpose
Implements all moving entity (mover) logic for Quake III Arena's game module, including the push/collision system for movers and spawn functions for doors, platforms, buttons, trains, and decorative movers (rotating, bobbing, pendulum, static).

## Core Responsibilities
- Execute per-frame movement for mover entities via `G_RunMover` / `G_MoverTeam`
- Push (or block) entities that intersect a moving brush, with full rollback on failure
- Manage binary mover state transitions (POS1 ↔ POS2) and associated sounds/events
- Spawn and configure all `func_*` mover entity types from map data
- Handle door trigger volumes, spectator teleportation through doors, and platform touch logic
- Synchronize team-linked mover slaves so all parts move atomically

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `g_entities[]`, `level` (level_locals_t), `BG_EvaluateTrajectory`, `RadiusFromBounds`, `AngleVectors`, `VectorInverse`, `trap_*` syscalls, `G_Damage`, `G_AddEvent`, `G_UseTargets`, `G_Find`, `G_Spawn`, `G_FreeEntity`, `TeleportPlayer`, `Team_DroppedFlagThink`, `G_ExplodeMissile`, `G_RunThink`, `g_gravity` (vmCvar_t)

# code/game/g_public.h
## File Purpose
Defines the public interface contract between the Quake III game module (QVM) and the server engine. It declares server-visible entity flags, shared entity data structures, and the complete syscall tables for both engine-to-game (imports) and game-to-engine (exports) communication.

## Core Responsibilities
- Define `GAME_API_VERSION` for versioning the game/server ABI
- Declare `SVF_*` bitflags controlling server-side entity visibility and behavior
- Define `entityShared_t` and `sharedEntity_t` as the shared memory layout the server reads directly
- Enumerate all engine syscalls available to the game module (`gameImport_t`)
- Enumerate all entry points the server calls into the game module (`gameExport_t`)
- Expose BotLib syscall ranges (200–599) as part of the game import table

## External Dependencies
- `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t`, `vec3_t`, `vmCvar_t`, `qboolean` — defined in `q_shared.h` / `bg_public.h` (game-shared layer)
- `gentity_t` — defined in `g_local.h`; `g_public.h` only sees it as a forward-referenced pointer target through `sharedEntity_t`
- Server engine — consumes this header to understand entity layout and dispatch the VM syscall tables
- BotLib — its full API surface is tunneled through the `gameImport_t` enum rather than direct linking

# code/game/g_rankings.c
## File Purpose
Implements the game-side interface to Quake III Arena's global online rankings system, collecting and submitting per-player statistics (weapon usage, damage, deaths, pickups, rewards) to an external ranking service via trap calls during and at the end of each match.

## Core Responsibilities
- Drive the rankings subsystem each server frame (init, poll, status management)
- Enforce ranked-game rules (kick bots, cap timelimit/fraglimit)
- Track and submit per-player combat statistics: shots fired, hits given/taken, damage, splash
- Report death events classified as frags, suicides, or hazard kills
- Report item pickups (weapons, ammo, health, armor, powerups, holdables)
- Report time spent with each weapon equipped
- Finalize and submit match-level metadata on game-over

## External Dependencies
- **Includes:** `g_local.h` (game entity/client types, level globals, all trap declarations), `g_rankings.h` (QGR_KEY_* constants, `GR_GAMEKEY`)
- **Defined elsewhere:** `trap_RankCheckInit`, `trap_RankBegin`, `trap_RankPoll`, `trap_RankActive`, `trap_RankUserStatus`, `trap_RankUserReset`, `trap_RankReportInt`, `trap_RankReportStr` — ranking system trap calls into the engine/VM syscall layer; `level` (`level_locals_t`), `g_entities[]` — game globals; `ClientSpawn`, `SetTeam`, `DeathmatchScoreboardMessage`, `OnSameTeam` — other game module functions; `GR_GAMEKEY` — game-key constant (defined elsewhere, not in the provided headers)

# code/game/g_rankings.h
## File Purpose
Defines a comprehensive set of numeric key constants used to report per-player and per-session statistics to a global online rankings/scoring backend. Each key encodes metadata about the stat's type, aggregation method, and category directly within its numeric value.

## Core Responsibilities
- Define all `QGR_KEY_*` constants for the rankings reporting system
- Encode stat semantics (report type, stat type, data type, calculation method, category) into each key's decimal digits
- Provide per-weapon stat keys for all 10 base weapons (Gauntlet through Grapple) plus unknowns
- Conditionally define `MISSIONPACK`-exclusive keys for Team Arena weapons, ammo, powerups, and holdables
- Provide keys for session metadata (hostname, map, gametype, limits)
- Provide keys for hazards, rewards, CTF events, and teammate interaction

## External Dependencies

- No includes.
- The key encoding scheme implies an external global rankings server/API (not defined here) that interprets the numeric key structure.
- `MISSIONPACK` macro defined externally (build system / project settings) to enable Team Arena extensions.

---

**Key encoding schema** (decoded from the header comment):

| Digit position | Meaning | Notable values |
|---|---|---|
| 10⁹ | Report type | 1=normal, 2=dev-only |
| 10⁸ | Stat type | 0=match, 1=single-player, 2=duel |
| 10⁷ | Data type | 0=string, 1=uint32 |
| 10⁶ | Calculation | 0=raw, 1=add, 2=avg, 3=max, 4=min |
| 10⁴–10⁵ | Category | 00=general, 02=weapon, 09=reward, 11=CTF, etc. |
| 10²–10³ | Sub-category | weapon index (×100) or item tier |
| 10⁰–10¹ | Ordinal | stat variant within category |

# code/game/g_session.c
## File Purpose
Manages persistent client session data in Quake III Arena's server-side game module. Session data survives across level loads and tournament restarts by serializing to and deserializing from cvars at shutdown/reconnect time.

## Core Responsibilities
- Serialize per-client session state to named cvars on game shutdown
- Deserialize per-client session state from cvars on reconnect
- Initialize fresh session data for first-time connecting clients
- Initialize the world session and detect gametype changes across sessions
- Write all connected clients' session data atomically at shutdown

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer` — engine syscall stubs (g_syscalls.c)
  - `PickTeam`, `BroadcastTeamChange` — defined in `g_client.c` / `g_cmds.c`
  - `Info_ValueForKey` — defined in `q_shared.c`
  - `va` — defined in `q_shared.c`
  - `level`, `g_gametype`, `g_teamAutoJoin`, `g_maxGameClients` — globals defined in `g_main.c`

# code/game/g_spawn.c
## File Purpose
Parses the map's entity string at level load time, translates key/value spawn variables into binary `gentity_t` fields, and dispatches each entity to its class-specific spawn function. It is the entry point for all server-side entity instantiation from BSP data.

## Core Responsibilities
- Read and store raw key/value token pairs from the BSP entity string (`G_ParseSpawnVars`)
- Provide typed accessors for spawn variables: string, float, int, vector (`G_SpawnString`, etc.)
- Map string field names to `gentity_t` struct offsets and write typed values (`G_ParseField`)
- Look up and invoke the correct spawn function by classname (`G_CallSpawn`)
- Process the `worldspawn` entity to apply global level settings (`SP_worldspawn`)
- Filter entities by gametype flags (`notsingle`, `notteam`, `notfree`, `notq3a`/`notta`, `gametype`)
- Drive the full entity spawning loop for an entire level (`G_SpawnEntitiesFromString`)

## External Dependencies
- `g_local.h` — `gentity_t`, `level_locals_t`, `FOFS`, all `g_*` cvars, all `trap_*` syscalls
- `bg_public.h` (via `g_local.h`) — `bg_itemlist`, `gitem_t`, gametype constants (`GT_*`)
- **Defined elsewhere:** `G_Spawn`, `G_FreeEntity`, `G_Alloc`, `G_SpawnItem`, `G_Error`, `G_Printf`, `G_LogPrintf`, `trap_GetEntityToken`, `trap_SetConfigstring`, `trap_Cvar_Set`, `Q_stricmp`, all `SP_*` spawn functions (defined in `g_misc.c`, `g_mover.c`, `g_trigger.c`, `g_target.c`, `g_items.c`, etc.)

# code/game/g_svcmds.c
## File Purpose
Implements server-console-only commands for the Quake III Arena game module, including IP-based packet filtering/banning and administrative commands such as entity listing, team forcing, and bot management dispatch.

## Core Responsibilities
- Maintain an in-memory IP filter list (`ipFilters[]`) for allow/deny packet filtering
- Parse and persist IP ban masks to/from the `g_banIPs` cvar string
- Provide `G_FilterPacket` to gate incoming connections against the filter list
- Expose `Svcmd_AddIP_f` / `Svcmd_RemoveIP_f` for runtime ban management
- Implement `ConsoleCommand` as the single dispatch entry point for all server-console commands
- Provide `ClientForString` helper to resolve a client by slot number or name

## External Dependencies
- **Includes:** `g_local.h` (which transitively brings in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `trap_Argv`, `trap_Argc`, `trap_Cvar_Set`, `trap_SendConsoleCommand`, `trap_SendServerCommand` — VM syscall stubs (`g_syscalls.c`)
  - `G_Printf`, `Com_Printf` — logging (`g_main.c` / engine)
  - `SetTeam` — `g_cmds.c`
  - `ConcatArgs` — `g_cmds.c` (declared but not defined here)
  - `Svcmd_GameMem_f` — `g_mem.c`; `Svcmd_AddBot_f`, `Svcmd_BotList_f` — `g_bot.c`; `Svcmd_AbortPodium_f` — `g_arenas.c`
  - `g_filterBan`, `g_banIPs`, `g_dedicated` — cvars declared in `g_local.h`, registered in `g_main.c`
  - `level`, `g_entities` — global game state (`g_main.c`)

# code/game/g_syscalls.c
## File Purpose
Implements the DLL-side system call interface for the game module, providing typed C wrapper functions around a single variadic `syscall` function pointer set by the engine at load time. This file is excluded from QVM builds, where `g_syscalls.asm` is used instead.

## Core Responsibilities
- Receive and store the engine's syscall dispatch function pointer via `dllEntry`
- Wrap every engine API call (file I/O, cvars, networking, collision, etc.) as typed C functions
- Bridge float arguments through `PASSFLOAT` to avoid ABI issues with variadic integer-only syscall conventions
- Expose the full BotLib/AAS API surface to game logic via trap functions
- Provide entity action (EA) wrappers for bot input simulation

## External Dependencies
- `code/game/g_local.h` — pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, and all game type/enum definitions
- `G_PRINT`, `G_ERROR`, `G_LOCATE_GAME_DATA`, `BOTLIB_*`, `G_TRACE`, etc. — syscall opcode enumerations defined in `g_public.h` / `botlib.h` (defined elsewhere)
- `gentity_t`, `playerState_t`, `trace_t`, `vmCvar_t`, `usercmd_t`, `pc_token_t` — defined elsewhere
- `QDECL` — calling convention macro, defined in `q_shared.h`

# code/game/g_target.c
## File Purpose
Implements all `target_*` entity types for Quake III Arena's server-side game logic. These are invisible map entities that perform actions (give items, print messages, play sounds, fire lasers, teleport players, etc.) when triggered by other entities or players.

## Core Responsibilities
- Register spawn functions (`SP_target_*`) for each target entity class
- Assign `use` callbacks that execute when the entity is triggered
- Implement delayed firing, score modification, and message broadcasting
- Manage looping/one-shot audio via `target_speaker`
- Operate a continuous damage laser (`target_laser`) with per-frame think logic
- Teleport activating players to a named destination entity
- Link `target_location` entities into a global linked list for HUD location display

## External Dependencies
- **`g_local.h`** — `gentity_t`, `level_locals_t`, `gclient_t`, all trap/utility declarations
- **Defined elsewhere:** `Touch_Item` (g_items.c), `Team_ReturnFlag` (g_team.c), `G_UseTargets`, `G_Find`, `G_PickTarget`, `G_SetMovedir`, `G_AddEvent`, `G_SoundIndex`, `G_SetOrigin` (g_utils.c), `TeleportPlayer` (g_misc.c), `G_Damage` (g_combat.c), `AddScore` (g_client.c), `G_TeamCommand` (g_utils.c), all `trap_*` syscalls

# code/game/g_team.c
## File Purpose
Implements all server-side team game logic for Quake III Arena, covering CTF flag lifecycle (pickup, drop, capture, return), team scoring, frag bonuses, player location tracking, spawn point selection, and MISSIONPACK obelisk/harvester mechanics.

## Core Responsibilities
- Manage CTF and One-Flag-CTF flag state (at base, dropped, taken, captured)
- Award frag bonuses for flag carrier kills, carrier defense, and base defense
- Broadcast team sound events on score changes, flag events, and obelisk attacks
- Track and broadcast team overlay info (health, armor, weapon, location) per frame
- Provide team spawn point selection for CTF game starts and respawns
- Handle obelisk entity lifecycle: spawning, regen, pain, death, respawn (MISSIONPACK)
- Register map spawn entities for CTF player/spawn spots and obelisks

## External Dependencies
- `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, `g_team.h`)
- **Defined elsewhere:** `AddScore`, `CalculateRanks`, `G_Find`, `G_TempEntity`, `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `RespawnItem`, `SelectSpawnPoint`, `SpotWouldTelefrag`, `trap_SetConfigstring`, `trap_SendServerCommand`, `trap_InPVS`, `trap_Trace`, `trap_LinkEntity`, `level` (global), `g_entities` (global), all `g_obelisk*` cvars.

# code/game/g_team.h
## File Purpose
Header file for the Quake III Arena team-based game mode (CTF and Missionpack variants). It defines scoring constants for Capture the Flag mechanics and declares the public interface for team logic used by the server-side game module.

## Core Responsibilities
- Declares CTF scoring bonus constants, conditionally compiled for MISSIONPACK vs. base Q3A balancing
- Declares geometric radius and timing constants for proximity-based bonus logic
- Declares grapple hook physics constants
- Exposes the public function interface for all team/CTF game logic to the rest of the game module

## External Dependencies
- `gentity_t`, `team_t`, `vec3_t`, `qboolean` — defined in `g_local.h` / `q_shared.h`
- `MISSIONPACK` — preprocessor define controlling two distinct scoring balance sets; defined at build time
- All function bodies defined in `g_team.c`

# code/game/g_trigger.c
## File Purpose
Implements all map trigger entities for Quake III Arena's server-side game module. Handles volume-based activation, jump pads, teleporters, hurt zones, and repeating timers that fire targets when players or entities interact with them.

## Core Responsibilities
- Initialize trigger brush entities with correct collision contents and server flags
- Implement `trigger_multiple`: repeatable volume trigger with optional team filtering and wait/random timing
- Implement `trigger_always`: fires targets once on map load, then frees itself
- Implement `trigger_push` / `target_push`: jump pad physics, computing launch velocity to hit a target apex
- Implement `trigger_teleport`: client-predicted teleport volumes, with optional spectator-only mode
- Implement `trigger_hurt`: damage zones with SLOW/SILENT/NO_PROTECTION/START_OFF flags
- Implement `func_timer`: a non-spatial, toggleable repeating timer that fires targets

## External Dependencies
- **`g_local.h`**: `gentity_t`, `gclient_t`, `level_locals_t` (`level`), `g_gravity`, `FRAMETIME`, `CONTENTS_TRIGGER`, `SVF_NOCLIENT`, `TEAM_RED/BLUE/SPECTATOR`, `ET_PUSH_TRIGGER`, `ET_TELEPORT_TRIGGER`, damage flags, `MOD_TRIGGER_HURT`
- **Defined elsewhere:** `G_UseTargets`, `G_PickTarget`, `G_FreeEntity`, `G_SetMovedir`, `G_Sound`, `G_SoundIndex`, `G_Damage`, `TeleportPlayer`, `BG_TouchJumpPad`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_SetBrushModel`, `crandom`, `G_SpawnFloat`, `G_Printf`

# code/game/g_utils.c
## File Purpose
Provides core utility functions for the Quake III Arena server-side game module, including entity lifecycle management (spawn, free, temp entities), entity search/targeting, event signaling, shader remapping, and miscellaneous math/string helpers.

## Core Responsibilities
- Entity allocation (`G_Spawn`), initialization (`G_InitGentity`), and deallocation (`G_FreeEntity`)
- Temporary event-entity creation (`G_TempEntity`)
- Entity search by field offset (`G_Find`) and random target selection (`G_PickTarget`)
- Target chain activation (`G_UseTargets`) and team-broadcast commands (`G_TeamCommand`)
- Game event attachment to entities (`G_AddEvent`, `G_AddPredictableEvent`)
- Shader remapping table management (`AddRemap`, `BuildShaderStateConfig`)
- Configstring index registration for models and sounds (`G_FindConfigstringIndex`)

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `g_entities[]`, `level` (`level_locals_t`), all `trap_*` syscall stubs, `G_Damage`, `BG_AddPredictableEventToPlayerstate`, `AngleVectors`, `VectorCompare`, `Com_sprintf`, `Q_stricmp`, `Q_strcat`, `SnapVector`

# code/game/g_weapon.c
## File Purpose
Implements all server-side weapon firing logic for Quake III Arena, translating player weapon inputs into world-space traces, damage events, and projectile spawns. It is the authoritative damage source for hitscan weapons and the launch point for projectile entities.

## Core Responsibilities
- Compute muzzle position and firing direction from player view state
- Execute hitscan traces for gauntlet, machinegun, shotgun, railgun, and lightning gun
- Spawn projectile entities for rocket, grenade, plasma, BFG, grapple (and MissionPack: nail, prox mine)
- Apply Quad Damage (and MISSIONPACK Doubler) multipliers to all outgoing damage
- Track per-client shot/hit accuracy counters; award "Impressive" for back-to-back railgun hits
- Emit temp entities (EV_BULLET_HIT_FLESH, EV_RAILTRAIL, EV_SHOTGUN, etc.) for client-side effects
- MISSIONPACK: handle Kamikaze holdable item with expanding radius damage and shockwave

## External Dependencies
- `g_local.h` → `q_shared.h`, `bg_public.h`, `g_public.h` (all game types and trap declarations)
- **Defined elsewhere:** `g_entities[]`, `level` (globals in `g_main.c`); `fire_rocket`, `fire_grenade`, `fire_plasma`, `fire_bfg`, `fire_grapple`, `fire_nail`, `fire_prox` (`g_missile.c`); `G_Damage`, `G_InvulnerabilityEffect` (`g_combat.c`); `OnSameTeam` (`g_team.c`); `g_quadfactor`, `g_gametype` (cvars registered in `g_main.c`); `trap_Trace`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_EntitiesInBox` (engine syscalls)

# code/game/inv.h
## File Purpose
A pure C header defining integer constants for inventory slots, item model indices, and weapon indices used by the bot AI system. It serves as a shared lookup table mapping game items to numeric identifiers consumed by botlib's fuzzy logic and goal-evaluation scripts.

## Core Responsibilities
- Defines `INVENTORY_*` slot indices for armor, weapons, ammo, powerups, and flags/cubes used by bot AI inventory queries
- Defines enemy awareness constants (`ENEMY_HORIZONTAL_DIST`, `ENEMY_HEIGHT`, `NUM_VISIBLE_*`) as pseudo-inventory fuzzy inputs
- Defines `MODELINDEX_*` constants that must stay synchronized with the `bg_itemlist` array in `bg_misc.c`
- Defines `WEAPONINDEX_*` constants mapping logical weapon slots to 1-based integer IDs

## External Dependencies
- **`bg_misc.c`** — `bg_itemlist[]` array ordering must exactly match the `MODELINDEX_*` sequence; a mismatch silently corrupts bot item recognition
- **`MISSIONPACK`** — conditional compilation guard present but body is empty (`#error` is commented out); mission pack items (`INVENTORY_KAMIKAZE`, `MODELINDEX_KAMIKAZE`, etc.) are defined unconditionally regardless of the guard


# code/game/match.h
## File Purpose
This header defines all symbolic constants used by the bot AI's natural-language chat matching and team-command messaging system. It provides message type identifiers, match-template context flags, command sub-type bitmasks, and variable-slot indices that map parsed chat tokens to structured bot commands.

## Core Responsibilities
- Define the escape character (`EC`) used to delimit in-game chat tokens
- Declare bitmask flags for match-template parsing contexts (e.g., CTF, teammate address, time)
- Enumerate all bot-to-bot and bot-to-player message type codes (`MSG_*`)
- Provide command sub-type bitmask flags (`ST_*`) for qualifying message semantics
- Define named indices for word-replacement variable slots in message templates

## External Dependencies
- No includes in this file itself.
- Consumed by: `code/game/ai_chat.c`, `code/game/ai_cmd.c`, `code/game/ai_team.c`, and related bot source files (defined elsewhere).
- `EC` (`"\x19"`) must match the escape character literal used in chat string definitions in `g_cmd.c` (comment-enforced contract, not compiler-enforced).

---

**Notes:**
- `ST_1FCTFGOTFLAG` (`65535` / `0xFFFF`) appears to be a sentinel or "all flags set" value rather than a single-bit flag — its use among power-of-two `ST_*` values suggests a special aggregate case for one-flag CTF mode.
- Several `#define` names collide in value (e.g., `THE_ENEMY` and `THE_TEAM` are both `7`; `FLAG` and `PLACE` are both `1`; `ADDRESSEE` and `MESSAGE` are both `2`) — these are intentional aliasing of variable-slot indices for different message contexts, not bugs.
- `MSG_WHOISTEAMLAEDER` contains a typo ("LAEDER" instead of "LEADER") preserved from the original id Software source.

# code/game/q_math.c
## File Purpose
Stateless mathematical utility library shared across all Quake III Arena modules (game, cgame, UI, renderer). Provides 3D vector math, angle conversion, plane operations, bounding box utilities, and fast approximation routines.

## Core Responsibilities
- Vector arithmetic: normalize, dot/cross product, rotate, scale, MA operations
- Angle utilities: conversion, normalization, interpolation, delta computation
- Plane operations: construction from points, sign-bit classification, box-plane side testing
- Bounding box management: clear, expand, radius computation
- Direction compression: float normal ↔ quantized byte index via `bytedirs` table
- Fast math approximations: `Q_rsqrt` (Quake fast inverse square root), `Q_fabs`
- Seeded PRNG: `Q_rand`, `Q_random`, `Q_crandom`

## External Dependencies
- **Includes**: `q_shared.h` (all type definitions, macros, inline variants)
- **Defined elsewhere**: `assert`, `sqrt`, `cos`, `sin`, `atan2`, `fabs`, `isnan` from `<math.h>`; `memcpy`, `memset` from `<string.h>`; `VectorNormalize` (called by `PerpendicularVector`, defined later in same file); `PerpendicularVector` (called by `RotatePointAroundVector`, defined later in same file — forward reference resolved at link time within TU)
- **Platform asm paths**: x86 MSVC `__declspec(naked)` `BoxOnPlaneSide`; Linux/FreeBSD i386 uses external asm (excluded via `#if` guard)

# code/game/q_shared.c
## File Purpose
A stateless utility library compiled into every Quake III code module (game, cgame, ui, botlib). It provides portable string handling, text parsing, byte-order swapping, formatted output, and info-string manipulation that must be available in all execution environments including the QVM.

## Core Responsibilities
- Clamping, path, and file extension utilities
- Byte-order swap primitives for cross-platform endianness handling
- Tokenizing text parser with comment stripping and line tracking
- Safe string library replacements (`Q_str*`, `Q_strncpyz`, etc.)
- Color-sequence-aware string utilities (`Q_PrintStrlen`, `Q_CleanStr`)
- `va()` / `Com_sprintf()` formatted print helpers
- Info-string key/value encoding, lookup, insertion, and removal

## External Dependencies
- `#include "q_shared.h"` — all type definitions, macros, and prototypes.
- `Com_Error`, `Com_Printf` — defined in `qcommon/common.c` (host side) or provided via syscall trap in VM modules.
- Standard C: `vsprintf`, `strncpy`, `strlen`, `strchr`, `strcmp`, `strcpy`, `strcat`, `atof`, `tolower`, `toupper`.

# code/game/q_shared.h
## File Purpose
The universal shared header included first by all Quake III Arena program modules (game, cgame, UI, botlib, renderer, and tools). It defines the engine's foundational type system, math library, string utilities, network-communicated data structures, and cross-platform portability layer. Mod authors must never modify this file.

## Core Responsibilities
- Cross-platform portability: compiler warnings, CPU detection, `QDECL`, `ID_INLINE`, `PATH_SEP`, byte-order swap functions
- Primitive type aliases (`byte`, `qboolean`, `qhandle_t`, `vec_t`, `vec3_t`, etc.)
- Math library: vector/angle/matrix macros and inline functions, `Q_rsqrt`, `Q_fabs`, bounding-box helpers
- String utilities: `Q_stricmp`, `Q_strncpyz`, color-sequence stripping, `va()`, `Com_sprintf`
- Engine data structures communicated over the network: `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `gameState_t`
- Cvar system interface: `cvar_t`, `vmCvar_t`, and all `CVAR_*` flag bits
- Collision primitives: `cplane_t`, `trace_t`, `markFragment_t`
- Info-string key/value API declarations
- VM compatibility: conditionally includes `bg_lib.h` instead of standard C headers when compiled for the Q3 virtual machine

## External Dependencies
- `bg_lib.h` — VM-only C standard library replacement (included conditionally)
- `surfaceflags.h` — `CONTENTS_*` and `SURF_*` bitmask constants shared with q3map
- Standard C headers (`assert.h`, `math.h`, `stdio.h`, `stdarg.h`, `string.h`, `stdlib.h`, `time.h`, `ctype.h`, `limits.h`) — native builds only
- **Defined elsewhere:** `ShortSwap`, `LongSwap`, `FloatSwap` (byte-order helpers in `q_shared.c`); `Q_rsqrt`, `Q_fabs` on x86 (`q_math.c`); all `extern vec3_t`/`vec4_t` globals (`q_shared.c`); `Hunk_Alloc`/`Hunk_AllocDebug` (engine hunk allocator); `Com_Error`, `Com_Printf` (implemented per-module in engine/game/cgame/ui)

# code/game/surfaceflags.h
## File Purpose
Defines bitmask constants for brush content types and surface properties shared across the game engine, tools (BSP compiler, bot library), and utilities. The comment explicitly states it must be kept identical in both the quake and utils directories.

## Core Responsibilities
- Define `CONTENTS_*` flags describing what a brush volume contains (solid, liquid, clip, portal, etc.)
- Define `SURF_*` flags describing per-surface rendering and gameplay properties
- Serve as a shared contract between the game module, renderer, collision system, bot library, and map compiler tools

## External Dependencies
- Mirrored (must stay in sync) in `code/game/q_shared.h` — the comment warns these definitions also need to be there.
- Referenced by: `code/qcommon/cm_load.c`, `code/game/bg_pmove.c`, `code/renderer/tr_*.c`, `code/botlib/be_aas_*.c`, `q3map/` compiler sources, `code/bspc/` sources.
- No includes — this file is a pure constant-definition leaf with no dependencies of its own.


# code/game/syn.h
## File Purpose
Defines bitmask constants for bot chat context flags used by the AI chat system. These flags identify the situational context in which a bot chat synonym or response is valid.

## Core Responsibilities
- Define a bitmask enumeration of chat/behavior contexts for the bot AI
- Distinguish team-specific contexts (CTF red/blue, Obelisk, Harvester)
- Provide a catch-all `CONTEXT_ALL` mask for context-agnostic entries

## External Dependencies
- No includes or external symbols. Standalone macro-only header.

---

**Notes on constants:**

| Constant | Value | Meaning |
|---|---|---|
| `CONTEXT_ALL` | `0xFFFFFFFF` | Matches any context |
| `CONTEXT_NORMAL` | `1` | Default/generic context |
| `CONTEXT_NEARBYITEM` | `2` | Bot is near an item |
| `CONTEXT_CTFREDTEAM` | `4` | CTF, red team |
| `CONTEXT_CTFBLUETEAM` | `8` | CTF, blue team |
| `CONTEXT_REPLY` | `16` | Replying to another chat message |
| `CONTEXT_OBELISKREDTEAM` | `32` | Overload gametype, red team |
| `CONTEXT_OBELISKBLUETEAM` | `64` | Overload gametype, blue team |
| `CONTEXT_HARVESTERREDTEAM` | `128` | Harvester gametype, red team |
| `CONTEXT_HARVESTERBLUETEAM` | `256` | Harvester gametype, blue team |
| `CONTEXT_NAMES` | `1024` | Context for name-specific synonyms |

Values are powers of two, designed to be OR-combined into a composite context mask for lookup and filtering.

# code/jpeg-6/jcapimin.c
## File Purpose
Implements the minimum application interface for the JPEG compression half of the IJG JPEG library. Provides the core lifecycle functions (create, destroy, abort) and essential compression control functions (finish, write marker, write tables) needed for both normal compression and transcoding scenarios.

## Core Responsibilities
- Initialize and zero a `jpeg_compress_struct`, set up the memory manager, and transition to `CSTATE_START`
- Destroy or abort a compression object by delegating to common routines
- Mark quantization and Huffman tables as sent or unsent (suppress/un-suppress)
- Drive any remaining multi-pass compression work and finalize the JPEG bitstream (write EOI, flush destination)
- Write arbitrary JPEG markers (COM/APPn) between `jpeg_start_compress` and the first scanline
- Write an abbreviated table-only JPEG datastream without image data

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMZERO`, `SIZEOF`)
- `jpeglib.h` / `jpegint.h` — all JPEG types, struct definitions, error codes
- `jinit_memory_mgr` — defined in `jmemmgr.c`
- `jinit_marker_writer` — defined in `jcmarker.c`
- `jpeg_abort`, `jpeg_destroy` — defined in `jcomapi.c`
- All `cinfo->master`, `cinfo->coef`, `cinfo->marker`, `cinfo->dest`, `cinfo->progress` method pointers — implemented in their respective submodule files

# code/jpeg-6/jcapistd.c
## File Purpose
Implements the standard JPEG compression API entry points for full-compression workflows: initializing a compression session, writing scanlines of image data, and writing raw downsampled data. Intentionally separated from `jcapimin.c` to prevent linking the full compressor into transcoding-only applications.

## Core Responsibilities
- Initialize a compression session and activate all encoder submodules
- Accept and process scanline-format image data from the caller
- Accept and process pre-downsampled (raw) image data in iMCU-row units
- Track and report scanline progress via an optional progress monitor hook
- Enforce call-sequence validity via `global_state` checks

## External Dependencies
- `jinclude.h` — system header portability layer (`MEMZERO`, `MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` — all public JPEG types and the `jpeg_compress_struct` definition
- `jpegint.h` (via `JPEG_INTERNALS`) — internal submodule interface structs (`jpeg_comp_master`, `jpeg_c_main_controller`, `jpeg_c_coef_controller`, etc.)
- `jerror.h` (via `JPEG_INTERNALS`) — `ERREXIT1`, `WARNMS`, `ERREXIT` macros and error codes
- **Defined elsewhere:** `jinit_compress_master` (jcmaster.c), `jpeg_suppress_tables` (jcparam.c), all vtable method implementations (`process_data`, `compress_data`, `pass_startup`, `prepare_for_pass`, `progress_monitor`, `init_destination`, `reset_error_mgr`)

# code/jpeg-6/jccoefct.c
## File Purpose
Implements the coefficient buffer controller for JPEG compression. It sits between the forward-DCT stage and entropy encoding, managing how DCT coefficient blocks are collected, buffered, and fed to the entropy encoder. It is the top-level controller of the JPEG compressor proper.

## Core Responsibilities
- Initialize and manage the coefficient buffer (single-MCU or full-image virtual arrays)
- Dispatch the correct `compress_data` function pointer based on pass mode
- Run forward DCT on input sample rows and accumulate coefficient blocks into MCUs
- Handle padding (dummy blocks) at right and bottom image edges
- Support single-pass (pass-through) and multi-pass (Huffman optimization / multi-scan) compression
- Suspend and resume mid-row if the entropy encoder stalls

## External Dependencies
- `jinclude.h` — system includes, SIZEOF, MEMZERO macros
- `jpeglib.h` / `jpegint.h` — `j_compress_ptr`, `jpeg_component_info`, `jvirt_barray_ptr`, `JBLOCKROW`, `J_BUF_MODE`, `JDIMENSION`, etc.
- **Defined elsewhere:** `jzero_far`, `jround_up` (utility routines); `cinfo->fdct->forward_DCT` (forward DCT module); `cinfo->entropy->encode_mcu` (entropy encoder); `cinfo->mem->*` (memory manager); `ERREXIT` (error handler macro)

# code/jpeg-6/jccolor.c
## File Purpose
Implements input colorspace conversion for the IJG JPEG compressor. It transforms application-supplied pixel data (RGB, CMYK, grayscale, YCbCr, YCCK) into the JPEG internal colorspace before encoding. This is the compression-side counterpart to `jdcolor.c`.

## Core Responsibilities
- Allocate and initialize lookup tables for fixed-point RGB→YCbCr conversion
- Convert interleaved RGB input rows to planar YCbCr output (most common path)
- Convert RGB rows to grayscale (Y-only)
- Convert CMYK rows to YCCK (inverts CMY, passes K through)
- Pass through grayscale or multi-component data unchanged (`null_convert`, `grayscale_convert`)
- Select and wire the correct conversion function pointer pair (`start_pass` + `color_convert`) during module initialization

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `JSAMPLE*` types, `jpeg_color_converter`, `J_COLOR_SPACE` enum, `JPOOL_IMAGE`, `ERREXIT`, `METHODDEF`, `GLOBAL`
- `jmorecfg.h` (via jpeglib) — `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE`, `INT32`
- `alloc_small` — defined in the JPEG memory manager (`jmemmgr.c`), called through `cinfo->mem`

# code/jpeg-6/jcdctmgr.c
## File Purpose
Manages the forward DCT (Discrete Cosine Transform) pipeline for JPEG compression. It selects the appropriate DCT algorithm at initialization, precomputes scaled quantization divisor tables per component, and drives the encode-time DCT-and-quantize step for each 8×8 sample block.

## Core Responsibilities
- Allocate and initialize the `my_fdct_controller` subobject and wire it into `cinfo->fdct`
- Select the active DCT routine (`jpeg_fdct_islow`, `jpeg_fdct_ifast`, or `jpeg_fdct_float`) based on `cinfo->dct_method`
- Precompute per-quantization-table divisor arrays (scaled and reordered from zigzag) during `start_pass`
- Load 8×8 pixel blocks into a workspace with unsigned-to-signed bias removal
- Invoke the chosen DCT routine in-place on the workspace
- Quantize/descale the 64 DCT coefficients and write them to the output coefficient block array

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, `JBLOCKROW`, `JSAMPARRAY`, `JCOEF`, `NUM_QUANT_TBLS`, `DCTSIZE2`, `JPOOL_IMAGE`
- `jdct.h` — `DCTELEM`, `forward_DCT_method_ptr`, `float_DCT_method_ptr`, `FAST_FLOAT`, fixed-point macros (`DESCALE`, `MULTIPLY16V16`, `SHIFT_TEMPS`)
- **Defined elsewhere:** `jpeg_fdct_islow` (`jfdctint.c`), `jpeg_fdct_ifast` (`jfdctfst.c`), `jpeg_fdct_float` (`jfdctflt.c`), `jpeg_zigzag_order` (IJG internal table), `ERREXIT`/`ERREXIT1` (error handler macros from `jerror.h`)

# code/jpeg-6/jchuff.c
## File Purpose
Implements Huffman entropy encoding for the IJG JPEG compression library. It handles both standard encoding (writing coded bits to the output stream) and a statistics-gathering pass used to generate optimal Huffman tables.

## Core Responsibilities
- Initialize and configure the Huffman encoder for a compression scan
- Build derived lookup tables (`c_derived_tbl`) from raw JHUFF_TBL definitions
- Emit variable-length Huffman codes and raw coefficient bits to the output buffer
- Encode one MCU's worth of DCT coefficient blocks (DC + AC) per JPEG Section F.1.2
- Handle output suspension and MCU-level rollback via `savable_state`
- Emit restart markers and reset DC predictions at restart boundaries
- Gather symbol frequency statistics and generate optimal Huffman tables (when `ENTROPY_OPT_SUPPORTED`)

## External Dependencies
- `jinclude.h` — portability macros (`MEMZERO`, `MEMCOPY`, `SIZEOF`)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_entropy_encoder`, `JHUFF_TBL`, `jpeg_component_info`, `jpeg_destination_mgr`
- `jchuff.h` — `c_derived_tbl`, shared extern declarations for `jpeg_make_c_derived_tbl` / `jpeg_gen_optimal_table`
- **Defined elsewhere**: `jpeg_natural_order` (zigzag index table, `jpegint.h`/`jutils.c`), `jpeg_alloc_huff_table`, all JPEG error codes (`JERR_*`), `ERREXIT`/`ERREXIT1` macros

# code/jpeg-6/jchuff.h
## File Purpose
Declares shared data structures and function prototypes for JPEG Huffman entropy encoding, used by both the sequential encoder (`jchuff.c`) and the progressive encoder (`jcphuff.c`).

## Core Responsibilities
- Define the `c_derived_tbl` structure representing a pre-computed Huffman encoding table
- Declare `jpeg_make_c_derived_tbl` for expanding a raw Huffman table into derived (ready-to-use) form
- Declare `jpeg_gen_optimal_table` for generating an optimal Huffman table from symbol frequency data
- Provide short-name aliases for linkers with limited external symbol length support

## External Dependencies
- `jpeglib.h` / `jpegint.h` — defines `j_compress_ptr`, `JHUFF_TBL`, `JPP`, `EXTERN`
- `jpeg_make_c_derived_tbl`, `jpeg_gen_optimal_table` — defined in `jchuff.c`

# code/jpeg-6/jcinit.c
## File Purpose
Performs master module selection and initialization sequencing for the JPEG compressor. It wires together all compression subsystems (preprocessing, DCT, entropy coding, coefficient buffering, etc.) by calling each module's `jinit_*` function in the correct order for a full compression pass.

## Core Responsibilities
- Invoke `jinit_c_master_control` to validate/process compression parameters
- Conditionally initialize preprocessing chain (color conversion, downsampling, prep controller) when raw data input is not used
- Initialize forward DCT module
- Select and initialize the correct entropy encoder (Huffman sequential, Huffman progressive, or error on arithmetic)
- Initialize coefficient and main controllers with appropriate buffering modes
- Initialize the JFIF/JPEG marker writer
- Trigger virtual array allocation via the memory manager
- Write the SOI (Start of Image) file header marker immediately

## External Dependencies
- `jinclude.h` — platform portability macros, system headers
- `jpeglib.h` — public JPEG API types; also pulls in `jpegint.h` and `jerror.h` (via `JPEG_INTERNALS`)
- **Defined elsewhere (called but not defined here):**
  - `jinit_c_master_control`, `jinit_color_converter`, `jinit_downsampler`, `jinit_c_prep_controller` (`jcmaster.c`, `jccolor.c`, `jcsample.c`, `jcprepct.c`)
  - `jinit_forward_dct` (`jcdctmgr.c`)
  - `jinit_huff_encoder`, `jinit_phuff_encoder` (`jchuff.c`, `jcphuff.c`)
  - `jinit_c_coef_controller`, `jinit_c_main_controller` (`jccoefct.c`, `jcmainct.c`)
  - `jinit_marker_writer` (`jcmarker.c`)
  - `ERREXIT` macro — defined in `jerror.h`

# code/jpeg-6/jcmainct.c
## File Purpose
Implements the main buffer controller for the JPEG compression pipeline. It sits between the pre-processor (downsampling/color conversion) and the DCT/entropy coefficient compressor, managing the intermediate strip buffer of downsampled JPEG-colorspace data.

## Core Responsibilities
- Allocate and manage per-component strip buffers (or optional full-image virtual arrays) to hold downsampled data
- Initialize pass state (iMCU row counters, buffer mode) at the start of each compression pass
- Drive the data flow loop: pull rows from the preprocessor into the strip buffer, then push complete iMCU rows to the coefficient compressor
- Handle compressor suspension (output-not-consumed) by backing up the input row counter and retrying on the next call
- Expose the `start_pass` and `process_data` method pointers on `jpeg_c_main_controller`

## External Dependencies
- `jinclude.h` — system includes, `MEMZERO`/`MEMCOPY`, `SIZEOF`
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `jpeg_c_main_controller`, `jpeg_component_info`, `JDIMENSION`, `JSAMPARRAY`, `J_BUF_MODE`, `DCTSIZE`, `MAX_COMPONENTS`, `jround_up`
- `ERREXIT` — error macro defined in `jerror.h`
- `cinfo->prep->pre_process_data` — defined in `jcprepct.c`
- `cinfo->coef->compress_data` — defined in `jccoefct.c`
- `cinfo->mem->alloc_small`, `alloc_sarray`, `request_virt_sarray`, `access_virt_sarray` — defined in `jmemmgr.c`

# code/jpeg-6/jcmarker.c
## File Purpose
Implements the JPEG marker writer module for the IJG JPEG compression library. It serializes all required JPEG datastream markers (SOI, SOF, SOS, DHT, DQT, DRI, APP0, APP14, EOI, etc.) to the output destination buffer.

## Core Responsibilities
- Emit raw bytes and 2-byte big-endian integers to the output destination
- Write quantization table markers (DQT) and Huffman table markers (DHT)
- Write frame header (SOFn) and scan header (SOS, DRI) markers
- Write file header (SOI + optional JFIF APP0 / Adobe APP14) and trailer (EOI)
- Write abbreviated table-only datastreams
- Initialize the `jpeg_marker_writer` vtable on `cinfo->marker`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMCOPY`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `jpeg_marker_writer`, `JQUANT_TBL`, `JHUFF_TBL`, `jpeg_component_info`, `jpeg_destination_mgr`
- `jerror.h` — `ERREXIT`, `ERREXIT1`, `TRACEMS` macros (defined elsewhere via error manager)
- `C_ARITH_CODING_SUPPORTED` — conditional compile guard for `emit_dac` body (defined in `jconfig.h`)

# code/jpeg-6/jcmaster.c
## File Purpose
Implements the master control logic for the IJG JPEG compressor. It handles parameter validation, initial image geometry setup, multi-scan script validation, and inter-pass sequencing (determining pass types and ordering for single-pass, Huffman-optimization, and multi-scan progressive compression).

## Core Responsibilities
- Validate image dimensions, sampling factors, and component counts before compression begins
- Compute per-component DCT block dimensions, downsampled sizes, and MCU layout
- Validate multi-scan scripts (including progressive JPEG spectral/successive-approximation parameters)
- Set up scan parameters and MCU geometry for each scan
- Drive the pass pipeline: dispatch `start_pass` calls to all active submodules in the correct order
- Track pass number, scan number, and pass type state across the full compression sequence
- Initialize and wire up the `jpeg_comp_master` vtable on the `cinfo` object

## External Dependencies
- `jinclude.h` — system include resolution, `SIZEOF`, `MEMCOPY` macros
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — all JPEG struct definitions, `JMETHOD`, `JPOOL_IMAGE`, `DCTSIZE`, `DCTSIZE2`, `MAX_COMPONENTS`, `MAX_COMPS_IN_SCAN`, `C_MAX_BLOCKS_IN_MCU`
- `jerror.h` — `ERREXIT`, `ERREXIT1`, `ERREXIT2` macros and error codes
- `jdiv_round_up` — defined elsewhere (jutils.c); integer ceiling division
- All submodule vtable objects (`cconvert`, `downsample`, `prep`, `fdct`, `entropy`, `coef`, `main`, `marker`) — defined and initialized in their respective source files

# code/jpeg-6/jcomapi.c
## File Purpose
Provides the shared application interface routines for the IJG JPEG library that are common to both compression and decompression paths. It implements object lifecycle management (abort and destroy) and convenience allocators for quantization and Huffman tables.

## Core Responsibilities
- Abort an in-progress JPEG operation without destroying the object, resetting it for reuse
- Fully destroy a JPEG object and release all associated memory
- Allocate and zero-initialize quantization table (`JQUANT_TBL`) instances
- Allocate and zero-initialize Huffman table (`JHUFF_TBL`) instances

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, system headers)
- `jpeglib.h` — defines `j_common_ptr`, `JQUANT_TBL`, `JHUFF_TBL`, `jpeg_memory_mgr`, pool constants (`JPOOL_PERMANENT`, `JPOOL_NUMPOOLS`), state constants (`DSTATE_START`, `CSTATE_START`)
- `jpeg_memory_mgr::free_pool`, `::self_destruct`, `::alloc_small` — defined elsewhere (implemented in `jmemmgr.c`)

# code/jpeg-6/jconfig.h
## File Purpose
Platform-specific configuration header for the JPEG-6 library, targeting Watcom C/C++ on MS-DOS or OS/2. It defines compiler/platform capability macros consumed by the rest of the libjpeg source tree.

## Core Responsibilities
- Advertises C language feature availability (prototypes, unsigned types, stddef/stdlib headers)
- Configures pointer model and string library preferences for the target platform
- Selects the default and fastest DCT (Discrete Cosine Transform) algorithm variant
- Conditionally enables supported image file formats for the standalone cjpeg/djpeg tools
- Guards internal-only settings (shift behavior) behind `JPEG_INTERNALS`

## External Dependencies
- No includes. Intended to be the first platform-adaptation header consumed by `jinclude.h`.
- `JDCT_FLOAT` — enum value defined in `jpeglib.h`; referenced here before that header is included, so order of inclusion matters.


# code/jpeg-6/jcparam.c
## File Purpose
Provides optional default-setting and parameter-configuration routines for the IJG JPEG compressor. Applications call these helpers to configure quantization tables, Huffman tables, colorspace, and encoding options before starting compression.

## Core Responsibilities
- Build and install scaled quantization tables from standard JPEG spec templates
- Convert user-friendly quality ratings (0–100) to quantization scale factors
- Install standard Huffman tables (DC/AC, luma/chroma) per JPEG spec section K.3
- Set all compressor defaults (quality 75, Huffman coding, no restart markers, etc.)
- Map input colorspace to JPEG output colorspace and configure per-component sampling
- Optionally generate a progressive JPEG scan script

## External Dependencies
- `jinclude.h` — system includes, `MEMCOPY`, `SIZEOF`, platform warning suppression
- `jpeglib.h` / `jpegint.h` / `jerror.h` (via `JPEG_INTERNALS`) — all struct definitions, constants, error macros
- **Defined elsewhere:** `jpeg_alloc_quant_table`, `jpeg_alloc_huff_table` (memory module); `ERREXIT`/`ERREXIT1`/`ERREXIT2` (error handler macros); `CSTATE_START`, `JPOOL_PERMANENT`, `BITS_IN_JSAMPLE`, `DCTSIZE2`, `MAX_COMPONENTS`, `NUM_ARITH_TBLS` (constants from jpegint.h/jconfig.h)

# code/jpeg-6/jcphuff.c
## File Purpose
Implements Huffman entropy encoding for progressive JPEG compression, handling all four scan types: DC initial, DC refinement, AC initial, and AC refinement passes. This is the progressive counterpart to the sequential Huffman encoder in `jchuff.c`.

## Core Responsibilities
- Initialize and configure the progressive entropy encoder per scan type
- Encode DC coefficient initial scans with point-transform and differential coding
- Encode AC coefficient initial scans with run-length and EOB-run coding
- Encode DC refinement scans (single bit per coefficient)
- Encode AC refinement scans with correction-bit buffering
- Collect symbol frequency statistics for optimal Huffman table generation
- Flush pending EOBRUN symbols and restart interval markers

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMZERO`, `SIZEOF`, etc.)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_component_info`, `JHUFF_TBL`, `jpeg_destination_mgr`, `JBLOCKROW`, scan params (`Ss`, `Se`, `Ah`, `Al`)
- `jchuff.h` — `c_derived_tbl`, `jpeg_make_c_derived_tbl`, `jpeg_gen_optimal_table`
- **Defined elsewhere:** `jpeg_natural_order` (zigzag-to-natural scan order table), `jpeg_alloc_huff_table`, `JPEG_RST0`, `JERR_*` error codes, `ERREXIT`/`ERREXIT1` macros

# code/jpeg-6/jcprepct.c
## File Purpose
Implements the JPEG compression preprocessing controller, which manages the pipeline stage between raw input scanlines and the downsampler. It orchestrates color conversion, intermediate buffering, and vertical edge padding to satisfy the downsampler's row-group alignment requirements.

## Core Responsibilities
- Initialize and own the `my_prep_controller` object attached to `cinfo->prep`
- Accept raw input scanlines and drive the color converter (`cinfo->cconvert->color_convert`)
- Buffer color-converted rows until a full row group is ready for downsampling
- Invoke the downsampler (`cinfo->downsample->downsample`) on complete row groups
- Pad the bottom edge of the image by replicating the last real pixel row
- Pad downsampler output to a full iMCU height at image bottom
- Optionally support context-row mode (for input smoothing), providing wraparound row-pointer buffers

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, system headers)
- `jpeglib.h` / `jpegint.h` — `jpeg_compress_struct`, `jpeg_component_info`, `jpeg_c_prep_controller`, `JSAMPARRAY`, `JSAMPIMAGE`, `JDIMENSION`, `JPOOL_IMAGE`, `DCTSIZE`
- `jcopy_sample_rows` — defined elsewhere (likely `jutils.c`); copies rows within a sample array
- `cinfo->cconvert->color_convert` — color space converter, defined elsewhere
- `cinfo->downsample->downsample` — downsampling module, defined elsewhere
- `ERREXIT`, `MIN` — macros from JPEG error/utility headers

# code/jpeg-6/jcsample.c
## File Purpose
Implements the downsampling module for the IJG JPEG compressor. It reduces the spatial resolution of color components (chroma subsampling) from the input image resolution down to the component's coded resolution before DCT processing.

## Core Responsibilities
- Provide per-component downsampling method dispatch via `sep_downsample`
- Implement box-filter downsampling for arbitrary integer ratios (`int_downsample`)
- Implement optimized 1:1 passthrough (`fullsize_downsample`)
- Implement optimized 2h1v and 2h2v downsampling with alternating-bias dithering
- Implement smoothed variants of 2h2v and fullsize downsampling (conditional on `INPUT_SMOOTHING_SUPPORTED`)
- Handle horizontal edge padding via `expand_right_edge`
- Select and wire up the appropriate per-component method pointer during init

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMCOPY`, etc.)
- `jpeglib.h` / `jpegint.h` — `j_compress_ptr`, `jpeg_component_info`, `jpeg_downsampler`, `JSAMPARRAY`, `JDIMENSION`, `INT32`, `GETJSAMPLE`, `JMETHOD`, `DCTSIZE`, `MAX_COMPONENTS`
- `jcopy_sample_rows` — defined elsewhere (jutils.c); bulk row copy
- `ERREXIT`, `TRACEMS` — error/trace macros expanding to `cinfo->err` method calls

# code/jpeg-6/jctrans.c
## File Purpose
Implements JPEG transcoding compression: writing pre-existing raw DCT coefficient arrays directly to an output JPEG file, bypassing the normal pixel-data compression pipeline. Also provides utilities for copying critical image parameters from a decompression source to a compression destination.

## Core Responsibilities
- Initialize a compress object for coefficient-based (transcoding) output via `jpeg_write_coefficients`
- Copy lossless-transcoding-safe parameters from a decompressor to a compressor via `jpeg_copy_critical_parameters`
- Select and wire up the minimal set of compression modules needed for transcoding (`transencode_master_selection`)
- Implement a specialized coefficient buffer controller that reads from pre-supplied virtual arrays instead of a pixel pipeline
- Generate on-the-fly dummy DCT padding blocks at image right/bottom edges during output

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — all JPEG object types, method interfaces, constants
- **Defined elsewhere:** `jpeg_suppress_tables`, `jpeg_set_defaults`, `jpeg_set_colorspace`, `jpeg_alloc_quant_table`, `jinit_c_master_control`, `jinit_huff_encoder`, `jinit_phuff_encoder`, `jinit_marker_writer`, `jzero_far`

# code/jpeg-6/jdapimin.c
## File Purpose
Implements the minimum public API for the JPEG decompression half of the IJG JPEG library. Provides object lifecycle management (create/destroy/abort), header reading, incremental input consumption, and decompression finalization routines.

## Core Responsibilities
- Initialize and zero-out a `jpeg_decompress_struct`, wiring up memory manager and input controller
- Destroy or abort a decompression object, releasing allocated resources
- Read and parse the JPEG header up to the first SOS marker via `jpeg_read_header`
- Drive the input state machine through `jpeg_consume_input`, handling DSTATE transitions
- Set default decompression parameters (colorspace, scaling, dithering, quantization)
- Install custom COM/APPn marker handler callbacks
- Finalize decompression (drain remaining input, release memory)

## External Dependencies
- **`jinclude.h`** — platform portability macros (`MEMZERO`, `SIZEOF`)
- **`jpeglib.h`** — all public JPEG types and struct definitions
- **`jpegint.h`** (via `JPEG_INTERNALS`) — internal module interfaces
- **`jerror.h`** (via `JPEG_INTERNALS`) — `ERREXIT1`, `WARNMS1`, `TRACEMS3` macros
- **Defined elsewhere:** `jinit_memory_mgr`, `jinit_marker_reader`, `jinit_input_controller` (module init functions); `jpeg_destroy`, `jpeg_abort` (`jcomapi.c`); `inputctl->consume_input`, `src->init_source`, `src->term_source`, `master->finish_output_pass` (subobject vtable methods)

# code/jpeg-6/jdapistd.c
## File Purpose
Implements the standard public API for the JPEG decompression pipeline, covering the full-decompression path from `jpeg_start_decompress` through scanline reading to buffered-image mode control. It is intentionally separated from `jdapimin.c` so that transcoder-only builds do not pull in the full decompressor.

## Core Responsibilities
- Initialize and drive the decompressor through its state machine (`DSTATE_*` transitions)
- Absorb multi-scan input into the coefficient buffer during startup
- Handle dummy output passes required by two-pass quantization
- Provide scanline-at-a-time output via `jpeg_read_scanlines`
- Provide raw iMCU-row output via `jpeg_read_raw_data`
- Manage buffered-image mode via `jpeg_start_output` / `jpeg_finish_output`

## External Dependencies
- `jinclude.h` — platform portability macros, system headers
- `jpeglib.h` — all public JPEG types and struct definitions; pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`
- `jinit_master_decompress` — defined in `jdmaster.c` (external)
- `cinfo->master->prepare_for_output_pass`, `finish_output_pass`, `is_dummy_pass` — implemented in `jdmaster.c`
- `cinfo->main->process_data` — implemented in `jdmainct.c`
- `cinfo->coef->decompress_data` — implemented in `jdcoefct.c`
- `cinfo->inputctl->consume_input`, `has_multiple_scans`, `eoi_reached` — implemented in `jdinput.c`
- `ERREXIT`, `ERREXIT1`, `WARNMS` — error macros resolving through `jerror.h` / `jdapimin.c`

# code/jpeg-6/jdatadst.c
## File Purpose
Implements a stdio-based JPEG compression data destination manager for the IJG JPEG library. It provides the output buffering and flushing logic that routes compressed JPEG bytes to a `FILE*` stream during encoding.

## Core Responsibilities
- Allocate and manage a 4096-byte output buffer for compressed JPEG data
- Flush the full buffer to disk via `fwrite` when it fills during compression
- Flush any remaining partial buffer bytes at end-of-compression
- Install the destination manager's three callback functions onto a `j_compress_ptr`
- Reuse an existing destination object if one is already attached to the compressor

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `JFWRITE` macro, `<stdio.h>`
- `jpeglib.h` — `j_compress_ptr`, `jpeg_destination_mgr`, `JOCTET`, `JPOOL_IMAGE`, `JPOOL_PERMANENT`, `boolean`
- `jerror.h` — `ERREXIT`, `JERR_FILE_WRITE`
- `fwrite`, `fflush`, `ferror` — C standard I/O (defined in `<stdio.h>`)
- `jpeg_start_compress`, `jpeg_finish_compress` — defined elsewhere; invoke the callbacks installed here

# code/jpeg-6/jdatasrc.c
## File Purpose
Implements a JPEG decompression data source manager that reads compressed JPEG data from an in-memory byte buffer (modified from the original stdio-based version). It satisfies the `jpeg_source_mgr` interface required by the IJG JPEG library.

## Core Responsibilities
- Provide a concrete `jpeg_source_mgr` implementation for memory-backed JPEG input
- Initialize and manage a fixed-size intermediate read buffer (`INPUT_BUF_SIZE = 4096`)
- Refill the decompressor's input buffer by copying from the in-memory source pointer
- Support skipping over unneeded data segments (APPn markers, etc.)
- Register all source manager callbacks on the `j_decompress_ptr` object via `jpeg_stdio_src`

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `memcpy` via `<string.h>`)
- `jpeglib.h` — `jpeg_source_mgr`, `j_decompress_ptr`, `JOCTET`, `JPOOL_PERMANENT`, `SIZEOF`, `jpeg_resync_to_restart`
- `jerror.h` — error/trace macros (included transitively; not directly used in this file)
- **Defined elsewhere:** `jpeg_resync_to_restart` (IJG library default restart-marker recovery)

# code/jpeg-6/jdcoefct.c
## File Purpose
Implements the coefficient buffer controller for JPEG decompression, sitting between entropy decoding and inverse-DCT stages. Manages both single-pass (streaming) and multi-pass (buffered-image, progressive) decompression modes, including optional interblock smoothing for progressive scans.

## Core Responsibilities
- Buffer MCU coefficient blocks received from the entropy decoder
- Drive the inverse-DCT (IDCT) transform per component block
- Coordinate input and output passes in multi-scan/buffered-image mode
- Implement JPEG K.8 interblock smoothing for progressive scans
- Initialize and wire up the `jpeg_d_coef_controller` vtable on startup

## External Dependencies
- `jinclude.h`, `jpeglib.h` (via `jpegint.h`, `jerror.h`)
- **Defined elsewhere:** `jzero_far`, `jcopy_block_row`, `jround_up`; all IDCT implementations (`inverse_DCT_method_ptr`); entropy decoder (`decode_mcu`); memory manager (`access_virt_barray`, `request_virt_barray`, `alloc_small`, `alloc_large`); input controller (`consume_input`, `finish_input_pass`)

# code/jpeg-6/jdcolor.c
## File Purpose
Implements output colorspace conversion for the IJG JPEG decompressor. It converts decoded JPEG component planes (YCbCr, YCCK, grayscale, CMYK) into the application's requested output colorspace (RGB, CMYK, grayscale, or pass-through).

## Core Responsibilities
- Build lookup tables for fixed-point YCbCr→RGB conversion coefficients
- Convert YCbCr component planes to interleaved RGB pixel rows
- Convert YCCK component planes to interleaved CMYK pixel rows
- Pass through grayscale (Y-only) data unchanged
- Perform null (same-colorspace) plane-to-interleaved reformatting
- Initialize and wire the `jpeg_color_deconverter` subobject into `cinfo`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMCOPY`
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_color_deconverter`, `JSAMPIMAGE`, `JSAMPARRAY`, `JDIMENSION`, `J_COLOR_SPACE` enum, `JPOOL_IMAGE`, `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE/PIXELSIZE`
- `jcopy_sample_rows` — defined in `jutils.c`
- `RIGHT_SHIFT`, `SHIFT_TEMPS`, `FIX` — macros from `jpegint.h`/`jmorecfg.h`
- `cinfo->mem->alloc_small` — memory manager defined in `jmemmgr.c`
- `cinfo->sample_range_limit` — populated by `jpeg_start_decompress` in `jdmaster.c`

# code/jpeg-6/jdct.h
## File Purpose
Private shared header for the JPEG DCT/IDCT subsystem within the Independent JPEG Group (IJG) library. It defines types, macros, and external declarations used by both the forward DCT (compression) and inverse DCT (decompression) modules and their per-algorithm implementation files.

## Core Responsibilities
- Define `DCTELEM` as the working integer type for forward DCT buffers (width depends on sample bit depth)
- Declare function pointer typedefs for forward DCT method dispatch (`forward_DCT_method_ptr`, `float_DCT_method_ptr`)
- Define per-algorithm multiplier types for IDCT dequantization tables (`ISLOW_MULT_TYPE`, `IFAST_MULT_TYPE`, `FLOAT_MULT_TYPE`)
- Provide the range-limiting macro `IDCT_range_limit` and `RANGE_MASK` for safe output clamping
- Declare all forward and inverse DCT routine entry points as `EXTERN`
- Supply fixed-point arithmetic macros (`FIX`, `DESCALE`, `MULTIPLY16C16`, `MULTIPLY16V16`) used across DCT implementation files
- Provide short-name aliases for linkers that cannot handle long external symbols

## External Dependencies
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JMETHOD`, `JPP`, `EXTERN`, `inverse_DCT_method_ptr`
- `jmorecfg.h` — `BITS_IN_JSAMPLE`, `MULTIPLIER`, `FAST_FLOAT`, `INT32`, `INT16`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- `RIGHT_SHIFT` — defined elsewhere (platform/compiler-specific macro, typically in `jconfig.h` or `jpegint.h`)
- `SHORTxSHORT_32`, `SHORTxLCONST_32` — optional compile-time flags defined by platform configuration; govern `MULTIPLY16C16` implementation

# code/jpeg-6/jddctmgr.c
## File Purpose
Manages the inverse DCT (IDCT) subsystem for JPEG decompression. It selects the appropriate IDCT implementation per component based on the requested DCT method and output scaling, and builds the corresponding dequantization multiplier tables used by the IDCT routines.

## Core Responsibilities
- Select the correct IDCT function pointer per component based on `DCT_scaled_size` and `dct_method`
- Convert raw zigzag-ordered quantization tables into method-specific multiplier tables (ISLOW, IFAST, FLOAT)
- Pre-zero multiplier tables so uninitialized components produce neutral gray output
- Cache the current IDCT method per component to avoid redundant table rebuilds
- Initialize and register the IDCT controller subobject with the decompressor

## External Dependencies
- `jinclude.h` — platform includes, `MEMZERO`, `SIZEOF`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, `J_DCT_METHOD` enum, `JPOOL_IMAGE`, `MAX_COMPONENTS`, `DCTSIZE`, `DCTSIZE2`
- `jdct.h` — `ISLOW_MULT_TYPE`, `IFAST_MULT_TYPE`, `FLOAT_MULT_TYPE`, `IFAST_SCALE_BITS`, `DESCALE`, `MULTIPLY16V16`, `SHIFT_TEMPS`, IDCT extern declarations
- **Defined elsewhere:** `jpeg_idct_islow`, `jpeg_idct_ifast`, `jpeg_idct_float`, `jpeg_idct_4x4`, `jpeg_idct_2x2`, `jpeg_idct_1x1` (individual IDCT implementation files); `jpeg_zigzag_order` (defined in `jutils.c`); `jpeg_inverse_dct` struct (defined in `jpegint.h`)

# code/jpeg-6/jdhuff.c
## File Purpose
Implements sequential (baseline) Huffman entropy decoding for the IJG JPEG library. It builds derived decoding tables from raw JHUFF_TBL data and decodes one MCU at a time from a compressed bitstream, supporting input suspension and restart marker handling.

## Core Responsibilities
- Build lookahead and min/max code tables from raw Huffman table data (`jpeg_make_d_derived_tbl`)
- Fill the bit-extraction buffer from the data source, handling FF/00 stuffing and end-of-data (`jpeg_fill_bit_buffer`)
- Decode a single Huffman symbol via slow-path bit-by-bit traversal when lookahead misses (`jpeg_huff_decode`)
- Decode one full MCU's DC and AC coefficients, writing dezigzagged output to `JBLOCKROW` (`decode_mcu`)
- Handle restart markers: flush bit buffer, re-read marker, reset DC predictors (`process_restart`)
- Initialize the entropy decoder module and wire up method pointers (`jinit_huff_decoder`)

## External Dependencies
- `jinclude.h` — platform includes, `MEMZERO`, `SIZEOF`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JHUFF_TBL`, `JBLOCKROW`, `JCOEF`
- `jdhuff.h` — `d_derived_tbl`, `bitread_*` types, `HUFF_DECODE`/`CHECK_BIT_BUFFER`/`GET_BITS` macros; shared with `jdphuff.c`
- `jpegint.h` (via `jpeglib.h` with `JPEG_INTERNALS`) — internal module structs
- `jerror.h` — `WARNMS`, `ERREXIT1`, warning/error codes
- **Defined elsewhere**: `jpeg_natural_order[]` (dezigzag table, used in `decode_mcu`); `cinfo->marker->read_restart_marker`; `cinfo->src->fill_input_buffer`

# code/jpeg-6/jdhuff.h
## File Purpose
Shared header for JPEG Huffman entropy decoding, providing derived table structures, bit-reading state types, and performance-critical inline macros used by both the sequential decoder (`jdhuff.c`) and progressive decoder (`jdphuff.c`).

## Core Responsibilities
- Define the `d_derived_tbl` structure for pre-computed Huffman lookup acceleration
- Define persistent and working bitreader state structures for MCU-boundary suspension support
- Provide `BITREAD_LOAD_STATE` / `BITREAD_SAVE_STATE` macros for register-level bit buffer management
- Expose `CHECK_BIT_BUFFER`, `GET_BITS`, `PEEK_BITS`, and `DROP_BITS` inline bit-extraction macros
- Expose the `HUFF_DECODE` macro implementing the fast lookahead decode path with slow fallback
- Declare the three out-of-line extern functions backing the macro fast paths

## External Dependencies
- `jpeglib.h` — `j_decompress_ptr`, `JHUFF_TBL`, `JOCTET`, `boolean`, `INT32`, `UINT8`, `JPP()`
- `jdhuff.c` — defines `jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`
- `jdphuff.c` — consumer of all three extern functions and all macros defined here

# code/jpeg-6/jdinput.c
## File Purpose
Implements the input controller module for the IJG JPEG decompressor. It orchestrates the state machine that alternates between reading JPEG markers (via `jdmarker.c`) and consuming compressed coefficient data (via the coefficient controller), dispatching to the appropriate submodule at each phase.

## Core Responsibilities
- Initialize the `jpeg_input_controller` subobject and wire up its method pointers
- Drive the marker-reading loop, detecting SOS and EOI markers
- Perform one-time image geometry setup on first SOS (`initial_setup`)
- Compute per-scan MCU layout for both interleaved and non-interleaved scans (`per_scan_setup`)
- Latch quantization tables at the start of each component's first scan (`latch_quant_tables`)
- Coordinate scan start/finish with the entropy decoder and coefficient controller
- Support full reset for re-use of a decompressor object

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` / `jpegint.h` — all JPEG types and submodule interfaces
- **Defined elsewhere:** `jdiv_round_up` (jutils.c), `cinfo->marker->read_markers` (jdmarker.c), `cinfo->entropy->start_pass` (jdhuff.c / jdphuff.c), `cinfo->coef->start_input_pass` / `consume_data` (jdcoefct.c), `cinfo->mem->alloc_small` (jmemmgr.c)

# code/jpeg-6/jdmainct.c
## File Purpose
Implements the main buffer controller for the JPEG decompressor, sitting between the coefficient decoder and the post-processor. It manages downsampled sample data in JPEG colorspace, optionally providing context rows (above/below neighbors) required by fancy upsampling algorithms.

## Core Responsibilities
- Allocate and manage the intermediate sample buffer between coefficient decode and post-processing
- Deliver iMCU row data to the post-processor as row groups
- Optionally maintain a "funny pointer" scheme to provide context rows without copying data
- Handle image top/bottom boundary conditions by duplicating edge sample rows
- Support a two-pass quantization crank mode that bypasses the main buffer entirely
- Initialize the `jpeg_d_main_controller` sub-object and wire it into `cinfo->main`

## External Dependencies
- **Includes:** `jinclude.h` (platform portability, `SIZEOF`, `MEMCOPY`), `jpeglib.h` (all JPEG types and sub-object interfaces, pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`)
- **Defined elsewhere:**
  - `jpeg_d_main_controller` (declared in `jpegint.h`)
  - `cinfo->coef->decompress_data` — coefficient controller
  - `cinfo->post->post_process_data` — post-processing controller
  - `cinfo->upsample->need_context_rows` — upsampler flag
  - `cinfo->mem->alloc_small`, `alloc_sarray` — memory manager
  - `ERREXIT`, `JPOOL_IMAGE`, `JBUF_PASS_THRU`, `JBUF_CRANK_DEST`, `METHODDEF`, `LOCAL`, `GLOBAL`, `JPP` — macros from `jpegint.h`/`jmorecfg.h`

# code/jpeg-6/jdmarker.c
## File Purpose
Implements JPEG datastream marker parsing for the IJG decompressor. It reads and decodes all standard JPEG markers (SOI, SOF, SOS, DHT, DQT, DRI, DAC, APP0, APP14, etc.) with full support for input suspension—if insufficient data is available, parsing aborts and resumes transparently on the next call.

## Core Responsibilities
- Scan the input stream for JPEG marker bytes (0xFF prefix sequences)
- Parse each marker's parameter segment and populate `j_decompress_ptr` fields
- Support suspendable I/O: return `FALSE` mid-parse if data runs out; resume on re-entry
- Install and dispatch per-marker handler function pointers (APPn, COM)
- Implement restart-marker synchronization and error recovery (`jpeg_resync_to_restart`)
- Initialize the `jpeg_marker_reader` subobject at decompressor creation time

## External Dependencies
- `jinclude.h` — system includes, `MEMCOPY`, `SIZEOF` macros
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_marker_reader`, `JHUFF_TBL`, `JQUANT_TBL`, `jpeg_component_info`, all `JPEG_*` status codes
- `jerror.h` — `ERREXIT`, `WARNMS2`, `TRACEMS*` macros
- **Defined elsewhere:** `jpeg_alloc_huff_table`, `jpeg_alloc_quant_table` (jcomapi.c); `datasrc->fill_input_buffer`, `skip_input_data`, `resync_to_restart` (source manager, e.g. jdatasrc.c)

# code/jpeg-6/jdmaster.c
## File Purpose
Master control module for the IJG JPEG decompressor. It selects which decompression sub-modules to activate, configures multi-pass quantization, and drives the per-pass setup/teardown lifecycle called by `jdapi.c`.

## Core Responsibilities
- Compute output image dimensions and DCT scaling factors (`jpeg_calc_output_dimensions`)
- Build the sample range-limit lookup table for fast pixel clamping (`prepare_range_limit_table`)
- Select and initialize all active decompressor sub-modules (IDCT, entropy decoder, upsampler, color converter, quantizer, buffer controllers)
- Manage per-output-pass start/finish sequencing and dummy-pass logic for 2-pass color quantization
- Expose `jinit_master_decompress` as the library entry point that boots the master object

## External Dependencies
- **Includes:** `jinclude.h`, `jpeglib.h` (pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`)
- **Defined elsewhere:** `jdiv_round_up` (jutils.c), `jinit_1pass_quantizer` (jquant1.c), `jinit_2pass_quantizer` (jquant2.c), `jinit_merged_upsampler` (jdmerge.c), `jinit_color_deconverter` (jdcolor.c), `jinit_upsampler` (jdsample.c), `jinit_d_post_controller` (jdpostct.c), `jinit_inverse_dct` (jddctmgr.c), `jinit_phuff_decoder` (jdphuff.c), `jinit_huff_decoder` (jdhuff.c), `jinit_d_coef_controller` (jdcoefct.c), `jinit_d_main_controller` (jdmainct.c)

# code/jpeg-6/jdmerge.c
## File Purpose
Implements a merged upsampling and YCbCr-to-RGB color conversion pass for JPEG decompression. By combining chroma upsampling and colorspace conversion into a single loop, it avoids redundant per-pixel multiplications for the shared chroma terms, yielding a significant throughput improvement for the common 2h1v and 2h2v chroma subsampling cases.

## Core Responsibilities
- Build precomputed integer lookup tables for YCbCr→RGB channel contributions from Cb and Cr
- Provide a `start_pass` routine that resets per-pass state (spare row, row counter)
- Dispatch upsampling via `merged_2v_upsample` (2:1 vertical) or `merged_1v_upsample` (1:1 vertical)
- Implement `h2v1_merged_upsample`: process one luma row, 2:1 horizontal chroma replication, emit one output row
- Implement `h2v2_merged_upsample`: process two luma rows, 2:1 horizontal and 2:1 vertical chroma replication, emit two output rows
- Manage a spare row buffer for the 2v case when the caller supplies only a single-row output buffer, and discard the dummy last row on odd-height images
- Register itself as `cinfo->upsample` during module initialization

## External Dependencies
- `jinclude.h` — platform portability, `SIZEOF`, `MEMCOPY`
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_upsampler`, `JSAMPIMAGE`, `JDIMENSION`, `INT32`, `JSAMPLE`, `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE`, `JPOOL_IMAGE`, `SHIFT_TEMPS`, `RIGHT_SHIFT`, `FIX`
- `jcopy_sample_rows` — defined in `jutils.c` (used in spare-row drain path)
- `use_merged_upsample` — defined in `jdmaster.c` (controls whether this module is selected)
- `cinfo->sample_range_limit` — populated by `jdmaster.c` startup

# code/jpeg-6/jdphuff.c
## File Purpose
Implements progressive JPEG Huffman entropy decoding for the IJG JPEG library. It handles all four scan types defined by the progressive JPEG standard: DC initial, DC refinement, AC initial, and AC refinement scans, with full support for input suspension (backtracking to MCU start on partial data).

## Core Responsibilities
- Initialize the progressive Huffman decoder state per scan pass (`start_pass_phuff_decoder`)
- Validate progressive scan parameters (Ss, Se, Ah, Al) and update coefficient progression status
- Decode DC coefficients for initial scans with delta-coding and bit-shifting
- Decode AC coefficients for initial scans including EOB run-length handling
- Decode DC/AC refinement scans (successive approximation bit-plane refinement)
- Handle restart markers and resynchronize decoder state
- Allocate and initialize the `phuff_entropy_decoder` object

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JBLOCKROW`, `JCOEF`, scan parameter fields
- `jdhuff.h` — `d_derived_tbl`, `bitread_perm_state`, `bitread_working_state`, `HUFF_DECODE`, `CHECK_BIT_BUFFER`, `GET_BITS`, `BITREAD_*` macros
- **Defined elsewhere:** `jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`, `jpeg_natural_order`, `ERREXIT*`/`WARNMS*` error macros

# code/jpeg-6/jdpostct.c
## File Purpose
Implements the JPEG decompression postprocessing controller, which manages the pipeline stage between upsampling/color-conversion and color quantization/reduction. It buffers decoded pixel data in either a single strip or a full-image virtual array depending on the quantization pass mode.

## Core Responsibilities
- Initialize and own the strip buffer or full-image virtual array used between upsample and quantize stages
- Select the correct processing function pointer (`post_process_data`) based on the current pass mode
- Drive the upsample→quantize pipeline for one-pass color quantization
- Buffer full-image rows during the first pass of two-pass color quantization (prepass, no output emitted)
- Re-read buffered rows and quantize+emit them during the second pass of two-pass quantization
- Short-circuit the postprocessing stage entirely when no color quantization is needed (delegate directly to upsampler)

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `jpeg_decompress_struct`, `jpeg_d_post_controller`, `jvirt_sarray_ptr`, `JSAMPIMAGE`, `JSAMPARRAY`, `JDIMENSION`, `J_BUF_MODE`, `JPOOL_IMAGE`
- **Defined elsewhere:** `jround_up` (math utility), `ERREXIT` (error macro), `cinfo->upsample->upsample`, `cinfo->cquantize->color_quantize`, `cinfo->mem->*` (memory manager vtable)

# code/jpeg-6/jdsample.c
## File Purpose
Implements the upsampling stage of the JPEG decompression pipeline. It expands chroma (and other subsampled) components back to full output resolution, optionally using bilinear ("fancy") interpolation or simple box-filter replication.

## Core Responsibilities
- Initialize the upsampler module and select per-component upsample methods during decompression setup
- Buffer one row group of upsampled data in `color_buf` before passing to color conversion
- Support multiple upsampling strategies: fullsize passthrough, no-op, integer box-filter, fast 2h1v/2h2v box, and fancy triangle-filter variants
- Track remaining image rows to handle images whose height is not a multiple of `max_v_samp_factor`
- Allocate intermediate color conversion buffers only for components that actually require rescaling

## External Dependencies
- `jinclude.h` — platform portability macros
- `jpeglib.h` / `jpegint.h` — JPEG decompressor structs, `jpeg_upsampler`, `jpeg_component_info`, sample typedefs
- `jcopy_sample_rows` — defined in `jutils.c`
- `jround_up` — defined in `jutils.c`
- `ERREXIT` — error macro from `jerror.h`

# code/jpeg-6/jdtrans.c
## File Purpose
Implements JPEG transcoding decompression — reading raw DCT coefficient arrays from a JPEG file without performing full image decompression. This is the "lossless" path used when transcoding (e.g., re-compressing with different parameters without quality loss).

## Core Responsibilities
- Provide `jpeg_read_coefficients()`, the public entry point for transcoding decompression
- Drive the input consumption loop to absorb the entire JPEG file into virtual coefficient-block arrays
- Initialize a minimal subset of decompressor modules sufficient for coefficient extraction (no IDCT, upsampling, color conversion, or quantization)
- Select and initialize the correct entropy decoder (Huffman sequential or progressive)
- Allocate and realize the full-image virtual coefficient buffer
- Initialize progress monitoring with scan-count estimates appropriate for transcoding

## External Dependencies
- **`jinclude.h`** — platform portability macros (`SIZEOF`, `MEMCOPY`, system headers)
- **`jpeglib.h`** — all public JPEG types and state machine structs; includes `jpegint.h` and `jerror.h` when `JPEG_INTERNALS` is defined
- **Defined elsewhere:**
  - `jinit_huff_decoder` — sequential Huffman decoder init (`jdhuff.c`)
  - `jinit_phuff_decoder` — progressive Huffman decoder init (`jdphuff.c`)
  - `jinit_d_coef_controller` — coefficient buffer controller init (`jdcoefct.c`)
  - `ERREXIT`, `ERREXIT1` — error macros resolving via `cinfo->err->error_exit`
  - `DSTATE_READY`, `DSTATE_RDCOEFS`, `DSTATE_STOPPING` — decompressor state constants (`jpegint.h`)

# code/jpeg-6/jerror.c
## File Purpose
This is a Quake III Arena-adapted version of the IJG JPEG library's error-handling module. It replaces the standard Unix `stderr`-based error output with Quake's renderer interface (`ri.Error` and `ri.Printf`), integrating JPEG decode/encode errors into the engine's error and logging systems.

## Core Responsibilities
- Define and populate the JPEG standard message string table from `jerror.h`
- Implement the `error_exit` handler that calls `ri.Error(ERR_FATAL, ...)` on fatal JPEG errors
- Implement `output_message` to route JPEG messages through `ri.Printf`
- Implement `emit_message` with warning-level filtering and trace-level gating
- Implement `format_message` to produce formatted error strings from message codes and parameters
- Implement `reset_error_mgr` to clear error state between images
- Provide `jpeg_std_error` to wire all handler function pointers into a `jpeg_error_mgr`

## External Dependencies
- `jinclude.h` — platform-specific includes and memory macros
- `jpeglib.h` — JPEG library types and struct definitions
- `jversion.h` — version string constants embedded in the message table
- `jerror.h` — message code enum and `JMESSAGE` macro (included twice via X-macro pattern)
- `../renderer/tr_local.h` — provides `ri` (`refimport_t`) for `ri.Error` and `ri.Printf`; **defined elsewhere** in the renderer module
- `jpeg_destroy` — defined elsewhere in the IJG library (`jcomapi.c`)
- `ri.Error`, `ri.Printf` — defined elsewhere; renderer import table populated at renderer initialization

# code/jpeg-6/jerror.h
## File Purpose
Defines all error and trace message codes for the IJG JPEG library as a `J_MESSAGE_CODE` enum, and provides a set of convenience macros for emitting fatal errors, warnings, and trace/debug messages through the JPEG library's error manager vtable.

## Core Responsibilities
- Declares the `J_MESSAGE_CODE` enum by expanding `JMESSAGE` macros into enum values
- Provides `ERREXIT`/`ERREXIT1–4`/`ERREXITS` macros for fatal error dispatch (calls `error_exit` function pointer)
- Provides `WARNMS`/`WARNMS1–2` macros for non-fatal/corrupt-data warnings (calls `emit_message` at level -1)
- Provides `TRACEMS`/`TRACEMS1–8`/`TRACEMSS` macros for informational and debug tracing (calls `emit_message` at caller-supplied level)
- Supports dual-inclusion pattern: first include builds the enum, second include (with `JMESSAGE` defined externally) builds a string table

## External Dependencies
- No `#include` directives in this file.
- `JCOPYRIGHT`, `JVERSION`: string macros, defined in `jversion.h` (included elsewhere).
- `JMSG_STR_PARM_MAX`: integer constant, defined in `jpeglib.h`.
- `j_common_ptr`, `j_compress_ptr`, `j_decompress_ptr`: typedefs defined in `jpeglib.h`.
- `strncpy`: standard C library, used in `ERREXITS` and `TRACEMSS`.
- `error_exit`, `emit_message`: function pointer fields on `jpeg_error_mgr`, defined/populated in `jerror.c`.

# code/jpeg-6/jfdctflt.c
## File Purpose
Implements the forward Discrete Cosine Transform (DCT) using floating-point arithmetic for the IJG JPEG library. It applies the Arai, Agui, and Nakajima (AA&N) scaled DCT algorithm to an 8×8 block of samples in-place, performing two separable 1-D passes (rows then columns).

## Core Responsibilities
- Accept a flat 64-element `FAST_FLOAT` array representing an 8×8 sample block
- Apply 1-D forward DCT across all 8 rows (Pass 1)
- Apply 1-D forward DCT down all 8 columns (Pass 2)
- Produce scaled DCT coefficients in-place (scaling deferred to quantization step)
- Guard entire implementation under `#ifdef DCT_FLOAT_SUPPORTED`

## External Dependencies
- `jinclude.h` — system include abstraction, `MEMZERO`/`MEMCOPY` macros
- `jpeglib.h` — JPEG library types (`FAST_FLOAT`, `DCTSIZE`, `GLOBAL`, etc.)
- `jdct.h` — DCT subsystem private declarations; declares `jpeg_fdct_float` extern and `float_DCT_method_ptr`
- `FAST_FLOAT` — defined elsewhere (in `jmorecfg.h` via `jpeglib.h`)
- `DCTSIZE` — defined as `8` in `jpeglib.h`
- `DCT_FLOAT_SUPPORTED` — compile-time feature flag, defined elsewhere (typically `jconfig.h`)

# code/jpeg-6/jfdctfst.c
## File Purpose
Implements the fast, lower-accuracy integer forward Discrete Cosine Transform (DCT) for the IJG JPEG library. It applies the Arai, Agui & Nakajima (AA&N) scaled DCT algorithm to an 8×8 block of DCT elements in-place, using only 5 multiplies and 29 adds per 1-D pass.

## Core Responsibilities
- Perform a 2-pass (row then column) separable 8-point 1-D forward DCT on a single 8×8 block
- Encode fixed-point multiplicative constants at 8 fractional bits (`CONST_BITS = 8`)
- Provide an optionally less-accurate descale path (`USE_ACCURATE_ROUNDING` not defined → plain right-shift)
- Guard the entire implementation behind `#ifdef DCT_IFAST_SUPPORTED`
- Write scaled DCT coefficients back into the input buffer in-place (output×8 convention per JPEG spec)

## External Dependencies
- `jinclude.h` — platform portability, `MEMZERO`/`MEMCOPY`, system headers
- `jpeglib.h` — top-level JPEG library types and constants (`DCTSIZE`, `DCTSIZE2`)
- `jdct.h` — `DCTELEM` typedef, `DESCALE`, `RIGHT_SHIFT`, `FIX`, `SHIFT_TEMPS`, forward DCT extern declarations
- `DCTELEM`, `INT32`, `RIGHT_SHIFT`, `SHIFT_TEMPS` — defined elsewhere (`jmorecfg.h`, `jdct.h`, compiler/platform headers)
- `DCT_IFAST_SUPPORTED` — configuration macro defined in `jconfig.h`

# code/jpeg-6/jfdctint.c
## File Purpose
Implements the slow-but-accurate integer forward Discrete Cosine Transform (FDCT) for the IJG JPEG library. It performs a separable 2-D 8×8 DCT using a scaled fixed-point version of the Loeffler-Ligtenberg-Moschytz algorithm with 12 multiplies and 32 adds per 1-D pass.

## Core Responsibilities
- Perform the forward DCT on a single 8×8 block of `DCTELEM` samples in-place
- Execute two separable 1-D DCT passes: first across all 8 rows, then all 8 columns
- Apply scaled fixed-point integer arithmetic to avoid floating-point at runtime
- Scale outputs by `sqrt(8) * 2^PASS1_BITS` after pass 1; remove `PASS1_BITS` scaling after pass 2, leaving a net factor-of-8 scale (consumed by the quantization step in `jcdctmgr.c`)

## External Dependencies
- `jinclude.h` — system include abstraction, `MEMZERO`/`MEMCOPY`, `<stdio.h>`, `<string.h>`
- `jpeglib.h` — `DCTSIZE`, `INT32`, `BITS_IN_JSAMPLE`, `JSAMPLE`, JPEG object types
- `jdct.h` — `DCTELEM`, `DESCALE`, `FIX`, `MULTIPLY16C16`, `CONST_SCALE`, `ONE`; declares `jpeg_fdct_islow` as `EXTERN`
- `SHIFT_TEMPS`, `RIGHT_SHIFT` — defined elsewhere (platform-specific, typically in `jmorecfg.h` or `jpegint.h`)
- `MULTIPLY16C16` — defined in `jdct.h`, platform-tunable for 16×16→32 multiply optimization
- `DCT_ISLOW_SUPPORTED` — compile-time feature flag, defined elsewhere in the build configuration

# code/jpeg-6/jidctflt.c
## File Purpose
Implements a floating-point inverse DCT (IDCT) with integrated dequantization for the IJG JPEG library. It converts an 8×8 block of quantized DCT coefficients back into pixel-domain sample values using the Arai, Agui, and Nakajima (AA&N) scaled DCT algorithm.

## Core Responsibilities
- Dequantize input JCOEF coefficients by multiplying against the component's float multiplier table
- Perform a separable 2-pass (column then row) 8-point floating-point IDCT
- Short-circuit columns where all AC terms are zero (DC-only optimization)
- Descale results by factor of 8 (2³) in the row pass
- Range-limit final values to valid `JSAMPLE` range via lookup table
- Write output samples into the caller-supplied output row buffer

## External Dependencies
- `jinclude.h` — platform includes (`stdio.h`, `string.h`, etc.) and utility macros
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `FLOAT_MULT_TYPE`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `FAST_FLOAT`
- `jmorecfg.h` (via `jpeglib.h`) — `FAST_FLOAT`, `MULTIPLIER`, `CENTERJSAMPLE`, `MAXJSAMPLE`, `SHIFT_TEMPS`
- `compptr->dct_table` — populated externally by `jddctmgr.c` during decompressor startup
- `cinfo->sample_range_limit` — populated externally by `jdmaster.c` (`prepare_range_limit_table`)

# code/jpeg-6/jidctfst.c
## File Purpose
Implements a fast, reduced-accuracy integer Inverse Discrete Cosine Transform (IDCT) for the IJG JPEG decompression library. It performs simultaneous dequantization and 8x8 block IDCT using the Arai, Agui & Nakajima (AA&N) scaled algorithm, trading precision for speed compared to the slow/accurate variant (`jidctint.c`).

## Core Responsibilities
- Dequantize 64 DCT coefficients using a precomputed multiplier table (`compptr->dct_table`)
- Execute a two-pass separable 1-D IDCT (columns first, then rows) on a single 8x8 block
- Short-circuit computation for columns/rows with all-zero AC terms (DC-only fast path)
- Scale and range-limit all 64 output pixels into valid `JSAMPLE` values (0–MAXJSAMPLE)
- Write one reconstructed 8x8 tile of pixel data into the caller-supplied output buffer

## External Dependencies
- `jinclude.h` — platform portability, `MEMZERO`/`MEMCOPY`, system headers
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `DCTELEM`, `IFAST_MULT_TYPE`, `IFAST_SCALE_BITS`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `FIX`
- `jmorecfg.h` (via jpeglib.h) — `BITS_IN_JSAMPLE`, `MULTIPLIER`, `INT32`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- **Defined elsewhere:** `IDCT_range_limit` result table populated by `jdmaster.c:prepare_range_limit_table`; `compptr->dct_table` populated by `jddctmgr.c`; `DCT_IFAST_SUPPORTED` guard defined in `jconfig.h`

# code/jpeg-6/jidctint.c
## File Purpose
Implements the slow-but-accurate integer inverse DCT (IDCT) for the IJG JPEG library, performing combined dequantization and 2D IDCT on a single 8×8 DCT coefficient block. This is the `JDCT_ISLOW` variant, based on the Loeffler–Ligtenberg–Moschytz algorithm with 12 multiplies and 32 adds per 1-D pass.

## Core Responsibilities
- Dequantize 64 DCT coefficients using the component's quantization multiplier table
- Execute a two-pass separable 2D IDCT (column pass then row pass) on an 8×8 block
- Apply scaled fixed-point arithmetic with compile-time integer constants to avoid floating-point at runtime
- Short-circuit all-zero AC columns/rows for a common-case speedup
- Range-limit and clamp all 64 output samples into valid `JSAMPLE` (0–255) values
- Write the decoded 8×8 pixel block into the caller-supplied output scanline buffer

## External Dependencies

- `jinclude.h` — platform includes, `MEMZERO`/`MEMCOPY`, `size_t`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `ISLOW_MULT_TYPE`, `DESCALE`, `FIX`, `MULTIPLY16C16`, `IDCT_range_limit`, `RANGE_MASK`, `CONST_SCALE`
- `jmorecfg.h` (via `jpeglib.h`) — `INT32`, `MULTIPLIER`, `BITS_IN_JSAMPLE`, `CENTERJSAMPLE`, `MAXJSAMPLE`, `SHIFT_TEMPS`, `RIGHT_SHIFT`
- **Defined elsewhere:** `IDCT_range_limit` table populated by `jdmaster.c:prepare_range_limit_table()`; multiplier table (`compptr->dct_table`) populated by `jddctmgr.c`

# code/jpeg-6/jidctred.c
## File Purpose
Implements reduced-size inverse DCT (IDCT) routines for JPEG decompression, producing 4x4, 2x2, or 1x1 pixel output from an 8x8 DCT coefficient block. These are used when downscaled image output is requested, avoiding a full 8x8 IDCT followed by downsampling.

## Core Responsibilities
- Dequantize DCT coefficients using the component's quantization table
- Perform a two-pass (column then row) reduced IDCT using simplified LL&M butterfly arithmetic
- Clamp output samples to valid range via a pre-built range-limit lookup table
- Write reduced-size pixel rows directly into the output sample buffer
- Short-circuit all-zero AC coefficient cases for performance

## External Dependencies

- `jinclude.h` — platform portability macros (`MEMZERO`, etc.)
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `ISLOW_MULT_TYPE`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `FIX`, `MULTIPLY16C16`, DCT size/precision constants
- `DCTSIZE`, `BITS_IN_JSAMPLE`, `INT32`, `MAXJSAMPLE`, `CENTERJSAMPLE` — defined elsewhere in the JPEG library configuration headers
- `RIGHT_SHIFT`, `SHIFT_TEMPS` — platform-specific shift helpers defined elsewhere

# code/jpeg-6/jinclude.h
## File Purpose
A portability header for the Independent JPEG Group (IJG) JPEG library that centralizes system include file selection and provides cross-platform abstraction macros. It resolves platform differences in standard library availability, string function families, and I/O operations so the rest of the JPEG library can use a uniform interface.

## Core Responsibilities
- Suppresses MSVC compiler warnings when building on Win32 targets
- Conditionally includes system headers (`stddef.h`, `stdlib.h`, `sys/types.h`, `stdio.h`) based on `jconfig.h` feature flags
- Abstracts BSD vs. ANSI/SysV string/memory functions (`bzero`/`bcopy` vs. `memset`/`memcpy`) behind `MEMZERO`/`MEMCOPY` macros
- Provides a `SIZEOF()` macro to guarantee `size_t` return from `sizeof()` on non-conforming compilers
- Provides `JFREAD`/`JFWRITE` macros wrapping `fread`/`fwrite` with portable argument casting

## External Dependencies
- `../jpeg-6/jconfig.h` — Watcom-targeted auto-configuration header; defines `HAVE_STDDEF_H`, `HAVE_STDLIB_H`, `NEED_BSD_STRINGS` absence, `NEED_SYS_TYPES_H` absence, and DCT defaults
- `<stddef.h>`, `<stdlib.h>`, `<sys/types.h>` — conditionally included per `jconfig.h` flags
- `<stdio.h>` — unconditionally included (required for `FILE` references in `jpeglib.h`)
- `<string.h>` or `<strings.h>` — selected at compile time for memory/string primitives
- `fread`, `fwrite`, `memset`/`memcpy` or `bzero`/`bcopy` — defined in the C standard library, not in this file

# code/jpeg-6/jload.c
## File Purpose
Provides a single function to load a JPEG image from the Quake III filesystem into a heap-allocated pixel buffer. It bridges the engine's virtual filesystem and memory allocator with the libjpeg-6 decompression API.

## Core Responsibilities
- Open a JPEG file via the engine's `FS_FOpenFileRead` filesystem abstraction
- Initialize and configure a libjpeg decompression object with standard error handling
- Drive the full libjpeg decompression pipeline (header read → start → scanline loop → finish → destroy)
- Allocate output pixel memory via engine's `Z_Malloc` (zone allocator)
- Return image dimensions and pixel data to the caller via out-parameters
- Close the file handle with `FS_FCloseFile` after decompression

## External Dependencies
- `../game/q_shared.h` — `fileHandle_t`, `qboolean`, `Z_Malloc`
- `../qcommon/qcommon.h` — `FS_FOpenFileRead`, `FS_FCloseFile`
- `jpeglib.h` (local jpeg-6) — entire libjpeg decompression API
- **Defined elsewhere:** `FS_FOpenFileRead`, `FS_FCloseFile` (filesystem module); `Z_Malloc` (zone memory, `qcommon/`); all `jpeg_*` functions (libjpeg-6 implementation files in `code/jpeg-6/`)

# code/jpeg-6/jmemansi.c
## File Purpose
Provides the ANSI-standard, system-dependent memory management backend for the IJG JPEG library. It implements allocation via standard `malloc`/`free` and temporary file backing store via POSIX `tmpfile()` for overflow when available memory is insufficient.

## Core Responsibilities
- Allocate and free "small" heap objects via `malloc`/`free`
- Allocate and free "large" heap objects (same mechanism on flat-memory systems)
- Report available memory to the JPEG memory manager
- Create, read, write, and close temporary backing-store files using `tmpfile()`
- Provide memory subsystem init/term hooks (trivial in this implementation)

## External Dependencies
- `jinclude.h` — platform includes, `JFREAD`/`JFWRITE` macros, `SIZEOF`
- `jpeglib.h` — `j_common_ptr`, `jpeg_common_struct`, `jpeg_memory_mgr`
- `jmemsys.h` — `backing_store_info`, `backing_store_ptr`, function prototypes
- `malloc`, `free` — C standard library heap (ANSI `<stdlib.h>`)
- `tmpfile`, `fseek`, `fclose` — C standard I/O (`<stdio.h>`)
- `ERREXIT`, `ERREXITS` — defined elsewhere in the JPEG library (`jerror.h` / `jmemmgr.c`); perform error exit via `cinfo->err->error_exit`

# code/jpeg-6/jmemdos.c
## File Purpose
MS-DOS-specific implementation of the IJG JPEG memory manager's system-dependent layer. Provides heap allocation and three types of backing store (DOS files, XMS extended memory, EMS expanded memory) for spilling JPEG working buffers when RAM is insufficient.

## Core Responsibilities
- Allocate and free small (near heap) and large (far heap) memory blocks
- Report available memory to the JPEG memory manager
- Select and generate unique temporary file names using the `TMP`/`TEMP` environment variables
- Open, read, write, and close DOS-file-based backing store via direct DOS calls (assembly stubs)
- Open, read, write, and close XMS (extended memory, V2.0) backing store via the XMS driver
- Open, read, write, and close EMS (expanded memory, LIM/EMS 4.0) backing store via the EMS driver
- Initialize and terminate the memory subsystem (`jpeg_mem_init`, `jpeg_mem_term`)

## External Dependencies
- `jinclude.h`, `jpeglib.h`, `jmemsys.h` — IJG JPEG library internals
- `<alloc.h>` (Turbo C) or `<malloc.h>` (MSVC) — far-heap routines
- `<stdlib.h>` — `malloc`, `free`, `getenv`
- Assembly stubs in `jmemdosa.asm` (defined elsewhere): `jdos_open`, `jdos_close`, `jdos_seek`, `jdos_read`, `jdos_write`, `jxms_getdriver`, `jxms_calldriver`, `jems_available`, `jems_calldriver`
- `ERREXIT`, `ERREXITS`, `TRACEMSS`, `TRACEMS1` — error/trace macros defined in `jerror.h` (via `jpeglib.h`)

# code/jpeg-6/jmemmgr.c
## File Purpose
Implements the system-independent JPEG memory manager for the IJG JPEG library. It provides pool-based allocation (small and large objects), 2-D array allocation for image samples and DCT coefficient blocks, and virtual array management with optional disk-backed overflow storage.

## Core Responsibilities
- Pool-based allocation and lifetime management of "small" and "large" memory objects across `JPOOL_PERMANENT` and `JPOOL_IMAGE` lifetimes
- Allocation of 2-D sample arrays (`JSAMPARRAY`) and coefficient-block arrays (`JBLOCKARRAY`) with chunked large-object backing
- Registration and deferred realization of virtual (potentially disk-backed) sample and block arrays
- Swapping virtual array strips between in-memory buffers and backing store on demand
- Tracking total allocated space and enforcing `max_memory_to_use` policy
- Teardown: freeing all pools (including closing backing-store files) and destroying the manager itself

## External Dependencies
- `jinclude.h`, `jpeglib.h`, `jmemsys.h`
- **Defined elsewhere:** `jpeg_get_small`, `jpeg_free_small`, `jpeg_get_large`, `jpeg_free_large`, `jpeg_mem_available`, `jpeg_open_backing_store`, `jpeg_mem_init`, `jpeg_mem_term` (system-dependent, e.g., `jmemnobs.c` or `jmemansi.c`), `jzero_far`, `ERREXIT`/`ERREXIT1` macros (error handler)

# code/jpeg-6/jmemname.c
## File Purpose
Implements the system-dependent portion of the IJG JPEG memory manager for systems that require explicit temporary file naming. It provides memory allocation, memory availability reporting, and backing-store (temp file) management using named temporary files on disk.

## Core Responsibilities
- Allocate and free small and large memory objects via `malloc`/`free`
- Report available memory to the JPEG memory manager
- Generate unique temporary file names (via `mktemp` or manual polling)
- Open, read, write, and close backing-store temp files
- Initialize and terminate the memory subsystem

## External Dependencies
- `jinclude.h` — platform stdio/string includes, `JFREAD`/`JFWRITE` macros
- `jpeglib.h` — `j_common_ptr`, `jpeg_memory_mgr`, `ERREXIT`, `TRACEMSS`
- `jmemsys.h` — `backing_store_ptr`, `backing_store_info`, function signatures
- `<errno.h>` — `ENOENT` (conditional, `NO_MKTEMP` path only)
- `malloc`, `free` — defined in `<stdlib.h>` or declared extern
- `mktemp`, `unlink`, `fopen`, `fclose`, `fseek` — defined in system libc

# code/jpeg-6/jmemnobs.c
## File Purpose
Provides the Quake III renderer-integrated system-dependent JPEG memory manager backend. It replaces standard `malloc`/`free` with the renderer's `ri.Malloc`/`ri.Free` allocator functions, ensuring JPEG memory operations go through the engine's tracked heap. Backing store (disk temp files) is explicitly unsupported.

## Core Responsibilities
- Implement `jpeg_get_small`/`jpeg_free_small` via `ri.Malloc`/`ri.Free`
- Implement `jpeg_get_large`/`jpeg_free_large` via the same allocator (no distinction between small/large)
- Report unlimited available memory to the JPEG library (`jpeg_mem_available`)
- Unconditionally error out if backing store is ever requested (`jpeg_open_backing_store`)
- Provide no-op init/term lifecycle stubs (`jpeg_mem_init`/`jpeg_mem_term`)

## External Dependencies
- `jinclude.h` — platform include shims, `SIZEOF`, `MEMCOPY`, etc.
- `jpeglib.h` — JPEG library types (`j_common_ptr`, `backing_store_ptr`, `ERREXIT`, `JERR_NO_BACKING_STORE`)
- `jmemsys.h` — declares the function signatures this file implements
- `../renderer/tr_local.h` — exposes `extern refimport_t ri`, providing `ri.Malloc` and `ri.Free`
- `ri` (`refimport_t`) — defined elsewhere in the renderer; this file depends on it being initialized before any JPEG operation occurs

# code/jpeg-6/jmemsys.h
## File Purpose
Defines the interface between the system-independent JPEG memory manager (`jmemmgr.c`) and its system-dependent backend implementations. It declares the contract that any platform-specific memory manager must fulfill, covering small/large heap allocation, available-memory querying, and backing-store (temp file/XMS/EMS) management.

## Core Responsibilities
- Declare small-heap allocation/free functions (`jpeg_get_small`, `jpeg_free_small`)
- Declare large-heap allocation/free functions (`jpeg_get_large`, `jpeg_free_large`)
- Declare available-memory query (`jpeg_mem_available`)
- Define the `backing_store_info` struct with vtable-style method pointers for temp-file I/O
- Declare backing-store lifecycle functions (`jpeg_open_backing_store`, `jpeg_mem_init`, `jpeg_mem_term`)
- Provide short-name aliases for linkers with limited symbol-length support (`NEED_SHORT_EXTERNAL_NAMES`)

## External Dependencies
- `jpeglib.h` / `jpegint.h` — for `j_common_ptr`, `JMETHOD`, `JPP`, `FAR`, `EXTERN` macros
- `jconfig.h` — supplies `USE_MSDOS_MEMMGR`, `NEED_SHORT_EXTERNAL_NAMES`, `MAX_ALLOC_CHUNK` overrides
- `<stdio.h>` — `FILE *` used in the non-DOS `backing_store_info` branch
- All declared functions are **defined elsewhere** in one of: `jmemansi.c`, `jmemnobs.c`, `jmemdos.c`, `jmemname.c` (selected at build time)

# code/jpeg-6/jmorecfg.h
## File Purpose
Platform-portability and capability configuration header for the Independent JPEG Group (IJG) JPEG library. It defines primitive typedefs, compile-time capability switches, and machine-dependent tuning macros used throughout the JPEG codec.

## Core Responsibilities
- Define `JSAMPLE` (pixel sample type) and `JCOEF` (DCT coefficient type) based on bit-depth setting
- Provide portable integer typedefs (`UINT8`, `UINT16`, `INT16`, `INT32`, `JDIMENSION`, `JOCTET`)
- Guard against `unsigned char` / `char` signedness portability issues via `GETJSAMPLE`/`GETJOCTET` macros
- Declare function-linkage macros (`METHODDEF`, `LOCAL`, `GLOBAL`, `EXTERN`)
- Enable/disable encoder and decoder feature modules at compile time
- Configure RGB scanline channel ordering and pixel stride
- Provide performance hints: `INLINE`, `MULTIPLIER`, `FAST_FLOAT`

## External Dependencies
- No includes of its own.
- Consumed by: `jpeglib.h`, and transitively all `j*.c` translation units in `code/jpeg-6/`.
- Conditioned on external macros: `HAVE_UNSIGNED_CHAR`, `HAVE_UNSIGNED_SHORT`, `CHAR_IS_UNSIGNED`, `NEED_FAR_POINTERS`, `XMD_H`, `HAVE_PROTOTYPES`, `HAVE_BOOLEAN`, `JPEG_INTERNALS`, `__GNUC__` — all expected to be set (or absent) by `jconfig.h` or the build system.

# code/jpeg-6/jpegint.h
## File Purpose
Internal header for the Independent JPEG Group's libjpeg-6 library, declaring the vtable-style module interfaces and initialization entry points used to wire together the JPEG compression and decompression pipelines. It is not intended for application-level inclusion.

## Core Responsibilities
- Define the `J_BUF_MODE` enum controlling pass-through vs. full-image buffering modes
- Declare global state machine constants (`CSTATE_*`, `DSTATE_*`) for compress/decompress lifecycle tracking
- Provide virtual dispatch structs (function-pointer tables) for every compression and decompression sub-module
- Declare all `jinit_*` module initializer prototypes that wire up the pipeline at startup
- Declare utility function prototypes (`jdiv_round_up`, `jcopy_sample_rows`, etc.)
- Define portable `RIGHT_SHIFT` macro with optional unsigned-shift workaround
- Provide short-name aliases (`NEED_SHORT_EXTERNAL_NAMES`) for linkers with symbol-length limits

## External Dependencies
- `jpeglib.h` — defines `j_compress_ptr`, `j_decompress_ptr`, `JSAMPARRAY`, `JBLOCKROW`, `JDIMENSION`, `jvirt_barray_ptr`, `jpeg_component_info`, `jpeg_marker_parser_method`, `JMETHOD`, `JPP`, `EXTERN`
- `jmorecfg.h` (transitively) — `INT32`, `boolean`, `FAR`, `MAX_COMPONENTS`
- Symbols defined elsewhere: all `jinit_*` bodies live in their respective `.c` modules (`jcmaster.c`, `jdmaster.c`, `jdmarker.c`, etc.); utility bodies in `jutils.c`

# code/jpeg-6/jpeglib.h
## File Purpose
This is the primary public API header for the Independent JPEG Group's (IJG) JPEG library version 6, bundled with Quake III Arena for texture/image decoding. It defines all data structures, type aliases, and function prototypes required by any application that compresses or decompresses JPEG images.

## Core Responsibilities
- Define the master compression (`jpeg_compress_struct`) and decompression (`jpeg_decompress_struct`) context objects
- Declare all public API entry points for the JPEG encode/decode pipeline
- Define supporting data types: quantization tables, Huffman tables, component descriptors, scan scripts
- Declare pluggable manager interfaces (error, memory, progress, source, destination)
- Provide JPEG standard constants (DCT block size, table counts, marker codes)
- Conditionally include internal headers (`jpegint.h`, `jerror.h`) when `JPEG_INTERNALS` is defined

## External Dependencies

- `code/jpeg-6/jconfig.h` — Platform/compiler configuration flags (`HAVE_PROTOTYPES`, `HAVE_UNSIGNED_CHAR`, `JDCT_DEFAULT`, etc.)
- `code/jpeg-6/jmorecfg.h` — Type definitions (`JSAMPLE`, `JCOEF`, `JOCTET`, `UINT8`, `UINT16`, `INT32`, `JDIMENSION`), linkage macros (`EXTERN`, `METHODDEF`, `LOCAL`)
- `code/jpeg-6/jpegint.h` — Internal submodule struct definitions (included only when `JPEG_INTERNALS` is defined)
- `code/jpeg-6/jerror.h` — Error/message code enum and `ERREXIT`/`WARNMS`/`TRACEMS` macros (included only when `JPEG_INTERNALS` is defined)
- All internal submodule structs (`jpeg_comp_master`, `jpeg_entropy_encoder`, `jpeg_inverse_dct`, etc.) are defined elsewhere (in `jpegint.h`) and referenced here only as forward-declared pointers

# code/jpeg-6/jpegtran.c
## File Purpose
A standalone command-line application for lossless JPEG transcoding. It reads a JPEG file as raw DCT coefficients and rewrites it with different encoding parameters (progressive, arithmetic coding, restart intervals, etc.) without a full decode/re-encode cycle.

## Core Responsibilities
- Parse command-line switches to configure a JPEG compression context
- Open input/output files (or fall back to stdin/stdout)
- Decompress source JPEG into DCT coefficient arrays (lossless read)
- Copy critical parameters from source to destination compressor
- Re-compress using DCT arrays directly, preserving image quality
- Clean up all JPEG objects and file handles on exit

## External Dependencies

- **`cdjpeg.h`** — IJG common application declarations; provides `keymatch`, `read_stdin`, `write_stdout`, `read_scan_script`, `start_progress_monitor`, `end_progress_monitor`, `enable_signal_catcher`, `READ_BINARY`, `WRITE_BINARY`, `TWO_FILE_COMMANDLINE`
- **`jversion.h`** — `JVERSION`, `JCOPYRIGHT` string macros
- **Defined elsewhere (IJG library):** `jpeg_create_decompress`, `jpeg_create_compress`, `jpeg_std_error`, `jpeg_read_header`, `jpeg_read_coefficients`, `jpeg_copy_critical_parameters`, `jpeg_write_coefficients`, `jpeg_finish_compress`, `jpeg_destroy_compress`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`, `jpeg_stdio_src`, `jpeg_stdio_dest`, `jpeg_simple_progression`, `j_compress_ptr`, `jvirt_barray_ptr`

# code/jpeg-6/jquant1.c
## File Purpose
Implements 1-pass color quantization for JPEG decompression, mapping full-color pixel data to a fixed colormap using equally spaced color values. Supports three dithering modes: none, ordered (Bayer 16×16), and Floyd-Steinberg error diffusion.

## Core Responsibilities
- Build an orthogonal colormap by dividing available colors across output components
- Precompute a `colorindex` lookup table mapping pixel values to nearest colormap entries (premultiplied by stride)
- Provide fast per-row pixel quantization (general N-component and 3-component fast paths)
- Implement ordered dithering using a static Bayer matrix
- Implement Floyd-Steinberg dithering with per-component error accumulation arrays
- Initialize and configure the `jpeg_color_quantizer` vtable registered on `cinfo->cquantize`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` / `jerror.h` — JPEG structs, `GETJSAMPLE`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `ERREXIT*`, `TRACEMS*`, `jzero_far`
- `QUANT_1PASS_SUPPORTED` — compile-time guard; entire file is conditionally compiled
- `jzero_far` — defined elsewhere (jutils.c)
- `alloc_small`, `alloc_large`, `alloc_sarray` — provided by JPEG memory manager, defined elsewhere

# code/jpeg-6/jquant2.c
## File Purpose
Implements 2-pass color quantization (color mapping) for the IJG JPEG decompressor. Pass 1 builds a color usage histogram; pass 2 maps each pixel to the nearest entry in a custom colormap derived via median-cut, with optional Floyd-Steinberg dithering.

## Core Responsibilities
- Accumulate a 3D RGB histogram during prescan (pass 1)
- Run Heckbert/median-cut box-splitting to select a representative colormap
- Build a lazy-filled inverse colormap (histogram reused as lookup cache)
- Map pixels to colormap entries without dithering (`pass2_no_dither`)
- Map pixels to colormap entries with Floyd-Steinberg dithering (`pass2_fs_dither`)
- Initialize and own the error-limiting table for F-S dithering
- Register itself as the `cquantize` subobject on `j_decompress_ptr`

## External Dependencies
- `jinclude.h` — system includes, `SIZEOF`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` / `jerror.h` — JPEG object definitions, `METHODDEF`, `LOCAL`, `GLOBAL`, `ERREXIT`, `TRACEMS1`, `jzero_far`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `GETJSAMPLE`, `MAXJSAMPLE`, `BITS_IN_JSAMPLE`
- `RGB_RED`, `RGB_GREEN`, `RGB_BLUE` — defined in `jmorecfg.h`; control component ordering macros
- `cinfo->mem->alloc_small/alloc_large/alloc_sarray` — defined in memory manager, called via function pointers

# code/jpeg-6/jutils.c
## File Purpose
Provides shared utility tables and helper functions used by both the JPEG compressor and decompressor. Contains the canonical DCT coefficient ordering tables and low-level memory copy/zero operations needed throughout the IJG JPEG library.

## Core Responsibilities
- Define the `jpeg_zigzag_order` table mapping natural-order DCT positions to zigzag positions
- Define the `jpeg_natural_order` table mapping zigzag positions back to natural order (with overflow-safe padding)
- Provide integer arithmetic helpers (`jdiv_round_up`, `jround_up`)
- Provide portable sample-array row copy (`jcopy_sample_rows`)
- Provide portable DCT coefficient block row copy (`jcopy_block_row`)
- Provide FAR-pointer-safe memory zeroing (`jzero_far`) for DOS/80x86 compatibility

## External Dependencies
- `jinclude.h` — provides `MEMCOPY`, `MEMZERO`, `SIZEOF`, platform include dispatch
- `jpeglib.h` — provides `JSAMPARRAY`, `JBLOCKROW`, `JCOEFPTR`, `JDIMENSION`, `DCTSIZE2`, `JCOEF`, `JSAMPLE`, `FAR`, `GLOBAL`
- `jconfig.h` / `jmorecfg.h` (via `jpeglib.h`) — provide `NEED_FAR_POINTERS`, `USE_FMEM`, type sizes
- `memcpy` / `memset` / `_fmemcpy` / `_fmemset` — defined in system `<string.h>` or DOS far-memory library

# code/jpeg-6/jversion.h
## File Purpose
Defines version and copyright identification macros for the Independent JPEG Group's (IJG) JPEG library version 6. It serves as the single authoritative source of version metadata for the library build.

## Core Responsibilities
- Declares the library version string (`JVERSION`)
- Declares the copyright notice string (`JCOPYRIGHT`)

## External Dependencies
- No includes.
- No external symbols.

---

**Notes:**
- `JVERSION` value is `"6  2-Aug-95"`, indicating JPEG library release 6, dated August 2, 1995.
- `JCOPYRIGHT` credits Thomas G. Lane and the Independent JPEG Group.
- This file is vendored into the Quake III Arena source tree as part of the embedded `jpeg-6` library used for JPEG texture loading (see `code/jpeg-6/jload.c`).

# code/macosx/CGMouseDeltaFix.h
## File Purpose
This header declares a small macOS-specific shim that wraps CoreGraphics mouse delta querying. It provides a stable interface for retrieving raw mouse movement deltas, likely working around a platform bug or behavioral inconsistency in the `CGGetLastMouseDelta` API on early macOS versions.

## Core Responsibilities
- Declare initialization routine for the mouse delta fix subsystem
- Declare the mouse delta query function used by the macOS input layer
- Import the `ApplicationServices` framework to expose `CGMouseDelta` and related CG types

## External Dependencies
- `<ApplicationServices/ApplicationServices.h>` — provides `CGMouseDelta`, CoreGraphics types; macOS-only framework
- Implementation body: `code/macosx/CGMouseDeltaFix.m` (Objective-C)
- Consumers: `code/macosx/macosx_input.m` (defined elsewhere)

# code/macosx/CGPrivateAPI.h
## File Purpose
Declares types, structures, and constants that mirror Apple's private CoreGraphics Server (CGS) API on macOS. This header enables Quake III's macOS port to hook into undocumented system-level event notification machinery, specifically to receive global mouse movement events outside of normal window focus.

## Core Responsibilities
- Define scalar primitive typedefs mirroring CGS internal integer/float types
- Declare the `CGSEventRecordData` union covering all macOS low-level event variants
- Declare the `CGSEventRecord` struct representing a complete raw system event
- Declare function pointer types for the private `CGSRegisterNotifyProc` notification registration API
- Define notification type constants for mouse-moved and mouse-dragged events

## External Dependencies
- `<CoreGraphics/CoreGraphics.h>` — implied; uses `CGPoint` without definition in this file
- `CGSRegisterNotifyProc` — **defined in a private Apple framework** (CoreGraphics private); not linked directly, expected to be resolved at runtime
- No standard C library headers included directly

# code/macosx/Q3Controller.h
## File Purpose
Declares the `Q3Controller` Objective-C class, which serves as the macOS application controller (NSObject subclass) for Quake III Arena. It acts as the AppKit-facing entry point that bridges the macOS application lifecycle into the engine's main loop.

## Core Responsibilities
- Declares the main application controller class for the macOS platform
- Exposes an Interface Builder outlet for a splash/banner panel
- Provides IBActions for clipboard paste and application termination requests
- Declares `quakeMain` as the engine entry point invoked from the macOS app

## External Dependencies
- `<AppKit/AppKit.h>` — AppKit framework (NSObject, NSPanel, IBOutlet, IBAction)
- `DEDICATED` — preprocessor macro defined externally to strip client-only UI code
- `Q3Controller.m` — implementation file (defined elsewhere)
- `Quake3.nib` — Interface Builder nib file that instantiates this controller and wires `bannerPanel` (defined elsewhere)

# code/macosx/macosx_display.h
## File Purpose
Public interface header for macOS display management in Quake III Arena. It declares functions for querying display modes, managing hardware gamma ramp tables, and fading/unfading displays during mode switches.

## Core Responsibilities
- Declare the display mode query function (`Sys_GetMatchingDisplayMode`)
- Declare gamma table storage and retrieval functions
- Declare per-display and all-display fade/unfade operations
- Declare display release cleanup

## External Dependencies
- `tr_local.h` — renderer types (`qboolean`, `glconfig_t`, etc.)
- `macosx_local.h` — `glwgamma_t`, `glwstate_t`, `CGDirectDisplayID`, `glw_state` global
- `ApplicationServices/ApplicationServices.h` (via `macosx_local.h`) — `CGDirectDisplayID`, Core Graphics display API
- Implementations defined in `macosx_display.m` (not visible here)

# code/macosx/macosx_glimp.h
## File Purpose
A minimal platform-specific header that sets up the OpenGL framework includes for the macOS renderer backend. It conditionally enables a CGL macro optimization path that bypasses per-call context lookups.

## Core Responsibilities
- Include the macOS OpenGL framework headers (`OpenGL/gl.h`, `OpenGL/glu.h`, `OpenGL/OpenGL.h`)
- Conditionally include `glext.h` if `GL_EXT_abgr` is not already defined
- Optionally enable `CGLMacro.h` mode to eliminate redundant CGL context lookups per GL call
- Expose the `cgl_ctx` alias into translation units that include this header under `USE_CGLMACROS`

## External Dependencies
- `<OpenGL/OpenGL.h>` — CGL and core GL types (Apple framework)
- `<OpenGL/gl.h>` — Standard OpenGL API (Apple framework)
- `<OpenGL/glu.h>` — OpenGL Utility Library (Apple framework)
- `<OpenGL/glext.h>` — GL extensions (Apple framework), guarded by `GL_EXT_abgr`
- `macosx_local.h` — Pulled in only under `USE_CGLMACROS`; provides `glw_state` (`glwstate_t`) and the `_cgl_ctx` field (`CGLContextObj`)
- `<OpenGL/CGLMacro.h>` — Apple CGL macro rewrite header, only under `USE_CGLMACROS`; defined elsewhere (Apple SDK)

# code/macosx/macosx_local.h
## File Purpose
This is the macOS platform-specific shared header for Quake III Arena, declaring the OpenGL window/display state, macOS-specific system function prototypes, and accessor macros for managing the OpenGL context across the macOS rendering and input subsystems.

## Core Responsibilities
- Declares `glwstate_t`, the central macOS OpenGL window state structure
- Exposes the global `glw_state` instance for use across macOS platform files
- Provides `OSX_*` macros for safe GL context get/set/clear operations
- Declares input system entry points (`macosx_input.m`)
- Declares system event and display utility functions (`macosx_sys.m`)
- Declares GL visibility/pause functions (`macosx_glimp.m`)
- Handles C/Objective-C/C++ compatibility via `#ifdef __cplusplus` guards

## External Dependencies
- `qcommon.h` — engine core types (`qboolean`, `sysEventType_t`, `fileHandle_t`, etc.)
- `<ApplicationServices/ApplicationServices.h>` — `CGRect`, `CGDirectDisplayID`, `CGGammaValue`, etc.
- `<OpenGL/CGLTypes.h>` — `CGLContextObj`
- `<Foundation/NSGeometry.h>` (Obj-C only) — `NSRect`
- `macosx_timers.h` — `OTStampList` / `OmniTimer` profiling API (conditional on `OMNI_TIMER`)
- `NSOpenGLContext`, `NSWindow`, `NSEvent` — Cocoa objects (forward-declared or void-typed for C++ compatibility)
- `glw_state`, `Sys_IsHidden`, `glThreadStampList` — defined in `macosx_glimp.m` / `macosx_sys.m`

# code/macosx/macosx_qgl.h
## File Purpose
Autogenerated macOS-specific header that wraps every standard OpenGL 1.x function in a `qgl`-prefixed inline shim. Each shim optionally logs the call to a debug file and/or checks for GL errors after the call, then forwards to the real `gl*` function. A block of `#define` macros at the end redirects all bare `gl*` names to the error-message symbols, forcing callers to use the `qgl*` versions.

## Core Responsibilities
- Provide `qgl*` inline wrappers for every OpenGL 1.x core function (~200+ functions)
- Conditionally log GL call parameters to a debug file when `QGL_LOG_GL_CALLS` is defined
- Conditionally call `QGLCheckError()` after each GL call when `QGL_CHECK_GL_ERRORS` is defined
- Track nested `glBegin`/`glEnd` depth via `QGLBeginStarted` to suppress error checks inside a primitive block
- Poison all bare `gl*` symbols via `#define gl* CALL_THE_QGL_VERSION_OF_gl*` to enforce use of wrappers
- Provide `_glGetError()` as an unguarded bypass to avoid infinite recursion inside `QGLCheckError`

## External Dependencies
- Standard OpenGL headers (implicit via including code): all `gl*` functions, GL types
- `QGLCheckError` — defined in a companion `.m`/`.c` file (`macosx_qgl.m` or similar)
- `QGLDebugFile()` — returns a `FILE*` for debug output; defined elsewhere
- `QGLLogGLCalls`, `QGLBeginStarted` — extern globals, defined in companion source

# code/macosx/macosx_timers.h
## File Purpose
Conditional header that exposes macOS-specific OmniTimer profiling instrumentation for Quake III Arena's renderer and collision subsystems. When `OMNI_TIMER` is not defined, all macros collapse to no-ops, making the profiling entirely compile-time optional.

## Core Responsibilities
- Define `OTSTART`/`OTSTOP` macros for push/pop-style hierarchical timer nodes
- Declare extern `OTStackNode*` globals representing named profiling points across the renderer and collision paths
- Declare the `InitializeTimers()` initialization function
- Provide a zero-cost stub path (empty macros) when `OMNI_TIMER` is undefined

## External Dependencies
- `<OmniTimer/OmniTimer.h>` — macOS/OmniGroup framework providing `OTStackNode`, `OTStackPush`, `OTStackPop`; not present in the open-source release
- All `OTStackNode*` definitions live in a corresponding `.m` implementation file (not in this header)

# code/null/mac_net.c
## File Purpose
A null/stub implementation of the Mac network layer for Quake III Arena. It provides non-functional placeholder implementations of the platform-specific network functions required by the engine, always returning failure or doing nothing.

## Core Responsibilities
- Provide a stub `NET_StringToAdr` that only resolves "localhost" to `NA_LOOPBACK`
- Provide a no-op `Sys_SendPacket` that discards all outgoing packet data
- Provide a stub `Sys_GetPacket` that always reports no incoming packets

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `netadr_t`, `netadrtype_t` (`NA_LOOPBACK`), `memset`
- `../qcommon/qcommon.h` — `msg_t`, `netadr_t`, `Sys_SendPacket`/`Sys_GetPacket` declarations
- `strcmp`, `memset` — C standard library (via `q_shared.h` includes)

# code/null/null_client.c
## File Purpose
Provides a null (stub) implementation of the client subsystem for use in dedicated server or headless builds where no actual client functionality is required. All functions have empty bodies or return safe default values.

## Core Responsibilities
- Satisfy the linker's demand for client API symbols in non-client builds
- Provide a no-op `CL_Init` that registers the `cl_shownet` cvar (minimum viable init)
- Return safe defaults (`qfalse`/`qtrue`) from boolean-returning stubs
- Allow the server-side codebase to compile and link without the full client module

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`
- `Cvar_Get` — defined in `code/qcommon/cvar.c`
- `cvar_t`, `netadr_t`, `msg_t`, `fileHandle_t`, `qboolean` — defined elsewhere in qcommon/game headers

# code/null/null_glimp.c
## File Purpose
This is a null/stub implementation of the platform-specific OpenGL import layer (`GLimp`) and the dynamic OpenGL function pointer loader (`QGL`) for Quake III Arena. It provides empty no-op bodies for all required renderer platform interface functions, intended for headless/server builds or porting scaffolding where no actual display is needed.

## Core Responsibilities
- Declare the OpenGL extension function pointers required by the renderer (WGL/ARB/EXT)
- Provide a no-op `GLimp_EndFrame` so the renderer can call buffer swap without crashing
- Provide a no-op `GLimp_Init` / `GLimp_Shutdown` for renderer lifecycle hooks
- Provide no-op `GLimp_EnableLogging` and `GLimp_LogComment` for debug logging stubs
- Provide a trivially succeeding `QGL_Init` / `QGL_Shutdown` for the OpenGL dynamic loader interface

## External Dependencies
- **`../renderer/tr_local.h`** — pulls in `qboolean`, `qtrue`, OpenGL types (`GLenum`), and the `GLimp_*` / `QGL_*` declarations that this file implements
- `GLenum`, `GLuint`, etc. — defined via OpenGL headers transitively included through `tr_local.h` → `qgl.h`
- All `GLimp_*` and `QGL_*` symbols are **declared in `tr_local.h`** and **defined here** as stubs; the real implementations live in `code/win32/win_glimp.c`, `code/unix/linux_glimp.c`, etc.

# code/null/null_input.c
## File Purpose
Provides a no-op (null) implementation of the platform input subsystem for Quake III Arena. All functions are empty stubs, used when building a headless/dedicated server or a platform-agnostic null client where no actual input handling is needed.

## Core Responsibilities
- Stub out `IN_Init` so the engine's input initialization path can be called safely with no effect
- Stub out `IN_Frame` so the per-frame input polling path executes without error
- Stub out `IN_Shutdown` so the input teardown path completes cleanly
- Stub out `Sys_SendKeyEvents` so the OS key-event pump is a no-op

## External Dependencies
- `../client/client.h` — pulls in the full client subsystem header (key types, `clientActive_t`, `kbutton_t`, input function declarations, etc.), though none of those symbols are actually used here.

**Defined elsewhere (symbols the real implementation would use):**
- `Key_Event` / `Com_QueueEvent` — engine key/event queue (defined in `cl_keys.c` / `common.c`)
- `cl.mouseDx`, `cl.mouseDy` — mouse delta accumulators in `clientActive_t` (defined in `cl_main.c`)
- Platform OS handles — not applicable in null build

# code/null/null_main.c
## File Purpose
A minimal null/stub system driver for Quake III Arena, intended to aid porting efforts to new platforms. It provides no-op or trivially forwarding implementations of all required `Sys_*` platform abstraction functions, and contains the program entry point.

## Core Responsibilities
- Provide a compilable stub for all `Sys_*` platform interface functions required by `qcommon`
- Implement the program entry point (`main`) that initializes the engine and runs the main loop
- Forward streamed file I/O to standard C `fread`/`fseek`
- Print fatal errors to stdout and terminate the process
- Serve as a minimal baseline for porting to platforms without a real system driver

## External Dependencies
- `<errno.h>`, `<stdio.h>` — standard C I/O and error codes
- `../qcommon/qcommon.h` — engine-wide common declarations; defines `Com_Init`, `Com_Frame`, and the full `Sys_*` interface contract
- **Defined elsewhere:** `Com_Init`, `Com_Frame` (in `qcommon/common.c`); all `Sys_*` signatures are declared in `qcommon.h` but the authoritative platform implementations live in `code/win32/`, `code/unix/`, `code/macosx/`

# code/null/null_net.c
## File Purpose
Provides a null (stub) implementation of the platform-specific networking layer for Quake III Arena. It is used in headless or minimal build configurations where real network I/O is not needed, implementing only loopback address resolution.

## Core Responsibilities
- Stub out `Sys_SendPacket` so packet transmission is a no-op
- Stub out `Sys_GetPacket` so packet reception always returns nothing
- Implement `NET_StringToAdr` with minimal support: only resolves `"localhost"` to `NA_LOOPBACK`; all other addresses fail

## External Dependencies
- `../qcommon/qcommon.h` — provides `netadr_t`, `netadrtype_t`, `msg_t`, `qboolean`, `NA_LOOPBACK`, and the declared signatures for `NET_StringToAdr`, `Sys_SendPacket`, and `Sys_GetPacket`
- `strcmp`, `memset` — C standard library (via `qcommon.h` transitively including `q_shared.h`)
- `Sys_SendPacket`, `Sys_GetPacket`, `NET_StringToAdr` — declared in `qcommon.h`; **defined here** as null implementations

# code/null/null_snddma.c
## File Purpose
Provides a null (no-op) implementation of the platform-specific sound DMA driver interface. It exists to allow Quake III Arena to compile and run without any audio hardware or audio subsystem, returning safe default values for all sound queries.

## Core Responsibilities
- Stub out `SNDDMA_*` lifecycle functions so the portable sound mixer has valid symbols to call
- Stub out higher-level `S_*` sound API functions to prevent crashes in headless or null-platform builds
- Return `qfalse`/`0` from all init/query functions to signal audio is non-functional

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h` (for `qboolean`, `sfxHandle_t`) and `snd_public.h` (for the sound API contract)
- `sfxHandle_t` — typedef defined in `snd_public.h` (defined elsewhere)
- `qboolean`, `qfalse` — defined in `q_shared.h` (defined elsewhere)

# code/q3_ui/keycodes.h
## File Purpose
Defines the `keyNum_t` enumeration mapping all recognized input sources (keyboard, mouse, joystick, aux) to integer key codes for use by the input and UI systems. It serves as the shared vocabulary for key event dispatch throughout the Q3 UI module.

## Core Responsibilities
- Enumerate all virtual key codes for keyboard special keys, function keys, numpad keys, mouse buttons, scroll wheel, joystick buttons, and auxiliary inputs
- Anchor low-ASCII printable/control keys at their ASCII values (TAB=9, ENTER=13, ESC=27, SPACE=32)
- Provide `K_LAST_KEY` as a sentinel/bounds-check value (must remain < 256)
- Define `K_CHAR_FLAG` bitmask to multiplex character events over the same key-event path

## External Dependencies
- No includes.
- `keyNum_t` values are consumed by: `KeyEvent` (defined elsewhere in the client/input layer), menu/UI event handlers (defined elsewhere in `q3_ui/`).
- `K_CHAR_FLAG` (value `1024`) is used by the menu code to distinguish char vs. key events — the or'ing logic lives outside this file.

# code/q3_ui/ui_addbots.c
## File Purpose
Implements the in-game "Add Bots" menu for Quake III Arena, allowing players to add AI bots to a running server session. It builds a scrollable list of available bots with skill level and team selection controls.

## Core Responsibilities
- Initialize and display the Add Bots menu UI with all interactive widgets
- Retrieve and alphabetically sort available bot names from the game's bot info database
- Scroll a paginated list of up to 7 bot names at a time
- Handle bot selection highlighting and dispatch the `addbot` server command on confirmation
- Pre-cache all menu art assets for rendering
- Adapt team options based on the current game type (FFA vs. team modes)

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types, trap syscalls)
- **Defined elsewhere:**
  - `UI_GetBotInfoByNumber`, `UI_GetNumBots` — `ui_gameinfo.c`
  - `Menu_Draw`, `Menu_AddItem`, `Menu_AddItem` — `ui_qmenu.c`
  - `UI_PushMenu`, `UI_PopMenu`, `UI_DrawBannerString`, `UI_DrawNamedPic` — `ui_atoms.c`
  - `trap_*` syscall wrappers — `ui_syscalls.c`
  - `Info_ValueForKey`, `Q_strncpyz`, `Q_stricmp`, `Com_Clamp` — `q_shared.c`

# code/q3_ui/ui_atoms.c
## File Purpose
Core UI module for Quake III Arena's legacy menu system (`q3_ui`), providing the foundational drawing primitives, menu stack management, input dispatch, and per-frame refresh logic used by all menu screens.

## Core Responsibilities
- Maintain and manage the menu stack (`UI_PushMenu`, `UI_PopMenu`, `UI_ForceMenuOff`)
- Dispatch keyboard and mouse input events to the active menu
- Draw proportional (bitmap font) strings in multiple styles (normal, banner, shadow, pulse, inverse, wrapped)
- Draw fixed-width strings with Quake color code support
- Provide 640×480 virtual-coordinate primitives (`UI_FillRect`, `UI_DrawRect`, `UI_DrawHandlePic`, etc.)
- Initialize and refresh the UI system each frame (`UI_Init`, `UI_Refresh`)
- Route console commands to specific menu entry points (`UI_ConsoleCommand`)

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- `trap_*` syscall wrappers (defined in `ui_syscalls.c`) — all renderer, sound, key, cvar, and cmd operations
- `Menu_Cache`, `Menu_Draw`, `Menu_DefaultKey`, `Menu_SetCursor` — defined in `ui_qmenu.c`
- `g_color_table`, `Q_IsColorString`, `ColorIndex` — defined in `q_shared.c`
- All `UI_*Menu()` and `*_Cache()` functions — defined in their respective `ui_*.c` files

# code/q3_ui/ui_cdkey.c
## File Purpose
Implements the CD Key entry menu for Quake III Arena's legacy UI system. It allows the player to enter, validate, and submit a 16-character CD key, integrating with the engine's CD key storage and verification syscalls.

## Core Responsibilities
- Initialize and lay out the CD Key menu using the `menuframework_s` widget system
- Render a custom owner-draw field displaying the CD key input with real-time format feedback
- Pre-validate the CD key format client-side (length + allowed character set)
- Store a confirmed key via `trap_SetCDKey` on acceptance
- Pre-populate the field from the engine via `trap_GetCDKey`, clearing it if verification fails
- Cache menu artwork shaders for reuse
- Expose public entry points (`UI_CDKeyMenu`, `UI_CDKeyMenu_f`, `UI_CDKeyMenu_Cache`) consumed by the rest of the UI module

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`
- **Defined elsewhere:**
  - `trap_SetCDKey`, `trap_GetCDKey`, `trap_VerifyCDKey` — engine syscall wrappers (`ui_syscalls.c`)
  - `trap_R_RegisterShaderNoMip`, `trap_Key_GetOverstrikeMode`, `trap_Cvar_Set` — engine syscall wrappers
  - `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — `ui_atoms.c` / `ui_qmenu.c`
  - `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`, `UI_DrawProportionalString` — `ui_atoms.c`
  - `uis` (`uiStatic_t`) — global UI state, `ui_atoms.c`
  - `color_yellow`, `color_orange`, `color_white`, `color_red`, `listbar_color` — `ui_qmenu.c`
  - `BIGCHAR_WIDTH`, `BIGCHAR_HEIGHT` — defined in shared UI headers

# code/q3_ui/ui_cinematics.c
## File Purpose
Implements the Cinematics menu for the Quake III Arena UI, allowing players to replay pre-rendered RoQ cutscene videos (id logo, intro, tier completions, and ending). It builds and presents a scrollable text-button list that triggers `disconnect; cinematic <name>.RoQ` commands when activated.

## Core Responsibilities
- Define and initialize all menu items for the Cinematics screen (banner, frame art, text buttons, back button)
- Gray out tier cinematic entries that the player has not yet unlocked via `UI_CanShowTierVideo`
- Handle back-navigation by popping the menu stack
- On item activation, set the `nextmap` cvar and issue a disconnect + cinematic playback command
- Handle the demo version special case for the "END" cinematic
- Expose a console-command entry point (`UI_CinematicsMenu_f`) that also repositions the cursor to a specific item
- Precache menu art shaders via `UI_CinematicsMenu_Cache`

## External Dependencies
- **`ui_local.h`** — pulls in all menu framework types, trap syscalls, `uis` global, `UI_CanShowTierVideo`, `UI_PopMenu`, `UI_PushMenu`, `va`, `color_red`, `color_white`, `QMF_*`, `QM_ACTIVATED`, `MTYPE_*`
- **Defined elsewhere:** `UI_CanShowTierVideo` (`ui_gameinfo.c`), `UI_PopMenu`/`UI_PushMenu` (`ui_atoms.c`), all `trap_*` syscall wrappers (`ui_syscalls.c`), `Menu_AddItem`/`Menu_SetCursorToItem` (`ui_qmenu.c`), `uis` global state (`ui_atoms.c`)

# code/q3_ui/ui_confirm.c
## File Purpose
Implements a reusable modal confirmation dialog and message box for the Quake III Arena legacy UI (q3_ui). It presents a yes/no prompt or a multi-line informational message overlaid on the current screen, invoking a callback with the user's boolean result.

## Core Responsibilities
- Display a modal yes/no confirmation dialog with a question string
- Display a modal message box with multiple text lines and a single "OK" button
- Route keyboard input (`Y`/`N`, arrow keys, tab) to the appropriate menu items
- Pop the menu from the stack and invoke a caller-supplied callback with the result
- Cache the confirmation frame artwork via `trap_R_RegisterShaderNoMip`
- Support an optional custom draw callback for additional overlay rendering
- Determine fullscreen vs. overlay mode based on connection state

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`)
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `Menu_DefaultKey`, `Menu_Draw`, `UI_DrawNamedPic`, `UI_DrawProportionalString`, `UI_ProportionalStringWidth`, `trap_R_RegisterShaderNoMip`, `trap_GetClientState`, `color_red`, key constants (`K_TAB`, `K_LEFTARROW`, etc.)

# code/q3_ui/ui_connect.c
## File Purpose
Renders the connection/loading screen shown while the client connects to a server. Handles display of connection state transitions, active file download progress, and ESC-key disconnection.

## Core Responsibilities
- Draw the full-screen connection overlay (background, server name, map name, MOTD)
- Display per-state status text (challenging, connecting, awaiting gamestate)
- Show real-time download progress: file size, transfer rate, estimated time remaining
- Track the last connection state to reset loading text on regression
- Handle the ESC key during connection to issue a disconnect command

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- **Defined elsewhere:** `trap_GetClientState`, `trap_GetConfigString`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cmd_ExecuteText`, `Menu_Cache`, `UI_SetColor`, `UI_DrawHandlePic`, `UI_DrawProportionalString`, `UI_DrawProportionalString_AutoWrapped`, `UI_ProportionalStringWidth`, `UI_ProportionalSizeScale`, `Info_ValueForKey`, `Com_sprintf`, `va`, `uis` (global `uiStatic_t`)

# code/q3_ui/ui_controls2.c
## File Purpose
Implements the full Controls configuration menu for Quake III Arena's legacy UI (q3_ui). It manages keyboard binding assignment, mouse/joystick configuration cvars, a live player model preview, and tabbed section navigation (Move/Look/Shoot/Misc).

## Core Responsibilities
- Define and manage the complete keybinding table (`g_bindings[]`) for all player actions
- Read current key bindings from the engine and populate local store (`Controls_GetConfig`)
- Write modified bindings and cvars back to the engine (`Controls_SetConfig`)
- Handle the "waiting for key" input capture state for rebinding
- Animate a 3D player model preview in response to focused action items
- Organize controls into four tabbed sections with dynamic show/hide of menu items
- Support resetting all bindings and cvars to defaults via a confirmation dialog

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h`, `keycodes.h`, `tr_types.h`
- **Defined elsewhere:** `trap_Key_*`, `trap_Cvar_*`, `trap_R_RegisterModel/Shader`, `trap_Cmd_ExecuteText` (syscall stubs in `ui_syscalls.c`); `UI_PlayerInfo_SetModel/SetInfo`, `UI_DrawPlayer` (`ui_players.c`); `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (`ui_qmenu.c`); `UI_ConfirmMenu` (`ui_confirm.c`); `bg_itemlist` (`bg_misc.c`)

# code/q3_ui/ui_credits.c
## File Purpose
Implements the credits screen menu for Quake III Arena's legacy UI (`q3_ui`). It renders a static list of id Software team members and pushes itself onto the menu stack as a fullscreen menu that quits the game on any keypress.

## Core Responsibilities
- Define and register the credits menu structure with the UI menu system
- Draw all credit text (roles and names) using proportional string rendering
- Handle key input by triggering a game quit command
- Push the credits screen onto the active menu stack as fullscreen

## External Dependencies
- **Includes:** `ui_local.h` → transitively pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`
- **Defined elsewhere:**
  - `UI_DrawProportionalString`, `UI_DrawString` — `ui_atoms.c`
  - `UI_PushMenu` — `ui_atoms.c`
  - `trap_Cmd_ExecuteText` — `ui_syscalls.c`
  - `color_white`, `color_red` — `ui_qmenu.c`
  - `menuframework_s`, `K_CHAR_FLAG`, `PROP_HEIGHT`, `PROP_SMALL_SIZE_SCALE`, `SMALLCHAR_HEIGHT` — `ui_local.h` / `q_shared.h`

# code/q3_ui/ui_demo2.c
## File Purpose
Implements the Demos menu for Quake III Arena's legacy UI module (`q3_ui`). It scans the `demos/` directory for demo files matching the current protocol version, populates a scrollable list, and allows the player to play a selected demo or navigate back.

## Core Responsibilities
- Initialize and lay out all widgets for the Demos menu screen
- Enumerate demo files via `trap_FS_GetFileList` filtered by protocol-versioned extension (e.g., `dm_68`)
- Strip file extensions and uppercase demo names for display
- Handle user interaction: play selected demo, navigate list left/right, go back
- Preload/cache all menu artwork shaders via `Demos_Cache`
- Guard against the empty-list degenerate case by disabling the "Go" button

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `trap_*` syscall wrappers (`ui_syscalls.c`)
  - `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` (`ui_atoms.c`)
  - `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (`ui_qmenu.c`)
  - `ScrollList_Key` (`ui_qmenu.c`)
  - `Q_stricmp`, `Q_strupr`, `Com_sprintf`, `va` (`q_shared.c`)
  - `color_white` (global color constant, `ui_qmenu.c`)

# code/q3_ui/ui_display.c
## File Purpose
Implements the Display Options menu screen within Quake III Arena's legacy UI module (`q3_ui`). It presents two hardware-facing sliders (brightness and screen size) alongside navigation tabs to sibling option screens (Graphics, Sound, Network).

## Core Responsibilities
- Initialize and lay out all widgets for the Display Options menu
- Pre-cache art assets (frame bitmaps, back button) at load time
- Map slider values to `r_gamma` and `cg_viewsize` cvars on activation
- Navigate to sibling option menus (Graphics, Sound, Network) via `UI_PopMenu` + push
- Gray out the brightness slider when the GPU does not support gamma (`uis.glconfig.deviceSupportsGamma`)
- Expose `UI_DisplayOptionsMenu` and `UI_DisplayOptionsMenu_Cache` as the public API for this screen

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `uis` (`uiStatic_t`) — global UI state, provides `glconfig.deviceSupportsGamma`
  - `Menu_AddItem`, `Menu_SetCursorToItem`, `UI_PushMenu`, `UI_PopMenu` — menu framework (`ui_qmenu.c` / `ui_atoms.c`)
  - `UI_GraphicsOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu` — sibling screen entry points
  - `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip` — VM syscall wrappers (`ui_syscalls.c`)
  - `color_red`, `color_white`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT` — shared UI constants

# code/q3_ui/ui_gameinfo.c
## File Purpose
Manages loading, parsing, and querying arena and bot metadata for the Quake III Arena UI module. Also tracks and persists single-player game progression, award data, and tier video unlock state via cvars.

## Core Responsibilities
- Load and parse arena info from `.arena` files and `scripts/arenas.txt` into a pool allocator
- Load and parse bot info from `.bot` files and `scripts/bots.txt`
- Assign ordered indices to arenas, separating single-player, special, and FFA arenas
- Query arena/bot records by number, map name, or special tag
- Read and write single-player scores per skill level via `g_spScores1–5` cvars
- Track award totals and tier cinematic unlock state via cvars
- Provide cheat/debug commands to unlock all levels and medals

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h` — shared types, `vmCvar_t`, `qboolean`, info-string API
- `COM_Parse`, `COM_ParseExt` — defined in `qcommon`
- `Info_SetValueForKey`, `Info_ValueForKey` — defined in `q_shared.c`
- `trap_*` syscalls — defined in `ui_syscalls.c`, dispatched into the engine VM interface
- `UI_SPLevelMenu_ReInit` — defined in `ui_spLevel.c`
- `uis` (`uiStatic_t`) — global UI state defined in `ui_atoms.c`

# code/q3_ui/ui_ingame.c
## File Purpose
Implements the in-game pause menu for Quake III Arena, presenting a vertical list of text buttons that allow the player to access game management options (team, bots, setup, server info, restart, quit, resume, leave) while paused mid-session.

## Core Responsibilities
- Define and initialize all menu items (`ingamemenu_t`) for the in-game overlay menu
- Conditionally gray out menu items based on runtime cvars (e.g., `sv_running`, `bot_enable`, `g_gametype`)
- Dispatch UI navigation events to the appropriate sub-menu or game command via `InGame_Event`
- Pre-cache the frame background shader via `InGame_Cache`
- Reset menu stack to top-level and push the initialized menu via `UI_InGameMenu`

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types)
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `UI_ConfirmMenu`, `UI_CreditMenu`, `UI_TeamMainMenu`, `UI_SetupMenu`, `UI_ServerInfoMenu`, `UI_AddBotsMenu`, `UI_RemoveBotsMenu`, `UI_TeamOrdersMenu`, `Menu_AddItem`, `trap_*` syscall wrappers, `uis` global, `color_red`, `Info_ValueForKey`

# code/q3_ui/ui_loadconfig.c
## File Purpose
Implements the "Load Config" UI menu for Quake III Arena, allowing the player to browse and execute `.cfg` configuration files found in the game's file system.

## Core Responsibilities
- Initializes and lays out the Load Config menu's UI widgets (banner, frame art, scrollable file list, navigation arrows, back/go buttons)
- Enumerates all `.cfg` files via the filesystem trap and populates a scrollable list
- Strips `.cfg` extensions and uppercases filenames for display
- Handles user interactions: executing the selected config, navigating the list, or dismissing the menu
- Pre-caches all menu art shaders via `UI_LoadConfig_Cache`

## External Dependencies
- **`ui_local.h`** — pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, all menu type definitions, and all `trap_*` syscall declarations
- **Defined elsewhere:** `trap_Cmd_ExecuteText`, `trap_FS_GetFileList`, `trap_R_RegisterShaderNoMip`, `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `ScrollList_Key`, `va`, `Q_stricmp`, `Q_strupr`, `color_white` — all resolved from other UI/qcommon translation units at link time

# code/q3_ui/ui_local.h
## File Purpose
Central internal header for the legacy `q3_ui` UI module. It declares all shared types, constants, extern variables, and function prototypes used across the UI subsystem's many `.c` source files.

## Core Responsibilities
- Define the menu-item type system (`MTYPE_*`) and flag bitmask (`QMF_*`) constants
- Declare all menu widget structs (`menuframework_s`, `menucommon_s`, `menufield_s`, `menuslider_s`, `menulist_s`, etc.)
- Declare the top-level UI state singleton `uiStatic_t uis`
- Expose `vmCvar_t` extern declarations for all UI-owned cvars
- Declare the full set of `trap_*` syscall wrappers used by UI VM code
- Forward-declare all per-screen cache/init/draw entry points across every UI screen file
- Declare the `playerInfo_t` / `lerpFrame_t` types used for 3D player preview rendering

## External Dependencies
- `game/q_shared.h` — core types (`vec3_t`, `qboolean`, `vmCvar_t`, `sfxHandle_t`, etc.)
- `cgame/tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t`)
- `ui/ui_public.h` — `uiExport_t`, `uiImport_t`, `uiMenuCommand_t`, `uiClientState_t` (imported from new UI; `UI_API_VERSION` overridden to 4)
- `keycodes.h` — `keyNum_t` enum, `K_CHAR_FLAG`
- `game/bg_public.h` — `weapon_t`, `animation_t`, `animNumber_t`, `MAX_ANIMATIONS`, game-type enums
- **Defined elsewhere:** All `trap_*` bodies (`ui_syscalls.c`), all `Menu_*` / `Bitmap_*` / `ScrollList_*` bodies (`ui_qmenu.c`), all per-screen `*_Cache` / `UI_*Menu` functions in their respective `.c` files.

# code/q3_ui/ui_login.c
## File Purpose
Implements the in-game login menu screen for Quake III Arena's online rankings system (GRank). It presents a modal dialog with name and password fields, wiring up input to the rankings authentication syscall.

## Core Responsibilities
- Define and initialize all UI widgets for the login form (frame, labels, text fields, buttons)
- Handle `LOGIN` and `CANCEL` button events via `Login_MenuEvent`
- Submit credentials to the rankings backend via `trap_CL_UI_RankUserLogin`
- Preload/cache the frame shader asset via `Login_Cache`
- Push the menu onto the UI stack via `UI_LoginMenu`

## External Dependencies
- **`ui_local.h`** — pulls in all menu types, trap wrappers, and helper declarations
- `trap_CL_UI_RankUserLogin` — defined in `ui_syscalls.c`/engine; submits credentials to the rankings server (not declared in the bundled header, implying it is a raw syscall wrapper unique to the GRank module)
- `trap_R_RegisterShaderNoMip` — renderer syscall
- `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` — defined in `ui_atoms.c`
- `Menu_AddItem` — defined in `ui_qmenu.c`
- `Rankings_DrawName`, `Rankings_DrawPassword` — owner-draw callbacks defined in `ui_rankings.c`

# code/q3_ui/ui_main.c
## File Purpose
This is the Q3 UI module's entry point for the QVM virtual machine. It implements `vmMain`, the sole gateway through which the engine dispatches commands into the UI module, and manages the registration and updating of all UI-related cvars.

## Core Responsibilities
- Expose `vmMain` as the single engine-facing entry point for all UI commands
- Route engine UI commands (init, shutdown, input events, refresh, menu activation) to the appropriate handler functions
- Declare all UI-side `vmCvar_t` globals that mirror engine cvars
- Define a `cvarTable_t` table mapping cvar structs to their name, default, and flags
- Implement `UI_RegisterCvars` and `UI_UpdateCvars` to batch-register and sync all cvars

## External Dependencies
- `ui_local.h` — aggregates `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`, all menu/subsystem declarations, and all `trap_*` syscall prototypes.
- `UI_Init`, `UI_Shutdown`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Refresh`, `UI_IsFullscreen`, `UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen` — defined in `ui_atoms.c` / other `q3_ui` files.
- `trap_Cvar_Register`, `trap_Cvar_Update` — defined in `ui_syscalls.c`; bridge to engine via QVM syscall ABI.
- `UI_API_VERSION` — defined as `4` in `ui_local.h` (overrides the value from `ui_public.h`).

# code/q3_ui/ui_menu.c
## File Purpose
Implements the Quake III Arena main menu screen, including menu item layout, 3D banner model rendering, error message display, and navigation to all top-level game sections.

## Core Responsibilities
- Initializes and configures the `mainmenu_t` menu item list at startup
- Handles menu item activation events, routing to sub-menus (SP, MP, Setup, Demos, Cinematics, Mods, Exit)
- Renders the 3D rotating banner model in the menu background using the renderer API
- Displays `com_errorMessage` as an overlay when the engine reports an error
- Conditionally shows the "TEAM ARENA" option when the `missionpack` mod directory exists
- Performs CD key validation on startup and redirects to the CD key menu if invalid
- Draws copyright/demo watermark strings at the bottom of the screen

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types)
- **Defined elsewhere:** `uis` (`uiStatic_t` global from `ui_atoms.c`), all `trap_*` syscall wrappers (`ui_syscalls.c`), `Menu_Draw`/`Menu_AddItem` (`ui_qmenu.c`), all `UI_*Menu()` navigation targets (their respective `.c` files), `color_red`/`menu_text_color`/`menu_null_sound` (`ui_qmenu.c`), `ui_cdkeychecked` (`ui_main.c`)

# code/q3_ui/ui_mfield.c
## File Purpose
Implements low-level editable text field widgets for the Q3 UI menu system. Provides both a raw `mfield_t` editing core and a higher-level `menufield_s` wrapper that integrates with the `menuframework_s` item system.

## Core Responsibilities
- Render a scrollable, optionally blinking text field with cursor (`MField_Draw`)
- Handle keyboard navigation: left/right arrows, Home, End, Delete, Insert (overstrike toggle)
- Handle character input with insert/overstrike modes and optional maxchars limit
- Clipboard paste via `trap_GetClipboardData`
- Initialize `menufield_s` bounding box geometry for hit-testing and layout
- Draw a `menufield_s` with focus highlight, label, and cursor arrow glyph
- Route menu-system key events to the underlying `mfield_t` with case/digit filtering

## External Dependencies
- **`ui_local.h`** — pulls in `mfield_t`, `menufield_s`, `menucommon_s`, key constants, draw style flags, `MAX_EDIT_LINE`, `QMF_*` flags, color externs.
- **Defined elsewhere:** `trap_GetClipboardData`, `trap_Key_GetOverstrikeMode`, `trap_Key_SetOverstrikeMode`, `trap_Key_IsDown`, `trap_Error`, `UI_DrawString`, `UI_DrawChar`, `UI_FillRect`, `Menu_ItemAtCursor`, `Q_islower`, `Q_isupper`, `Q_isalpha`, `menu_buzz_sound`, color arrays (`text_color_disabled`, `text_color_normal`, `text_color_highlight`, `listbar_color`).

# code/q3_ui/ui_mods.c
## File Purpose
Implements the Mods menu screen for Quake III Arena's legacy UI (`q3_ui`), allowing the player to browse installed game modifications and switch to one by setting `fs_game` and triggering a video restart.

## Core Responsibilities
- Enumerate available game mods via `trap_FS_GetFileList("$modlist", ...)`
- Populate a scrollable list UI widget with mod names and their directory names
- Handle "Go" action: write the selected mod's directory to `fs_game` cvar and execute `vid_restart`
- Handle "Back" action: pop the menu without making changes
- Pre-cache all menu artwork shaders on demand
- Register itself as a pushable menu via `UI_ModsMenu()`

## External Dependencies
- **`ui_local.h`** — menu framework types (`menuframework_s`, `menulist_s`, `menubitmap_s`, `menutext_s`, `menucommon_s`), all `trap_*` syscall declarations, `UI_PushMenu`/`UI_PopMenu`, `Menu_AddItem`, `Q_strncpyz`, `color_white`
- **Defined elsewhere:** `trap_Cvar_Set`, `trap_Cmd_ExecuteText`, `trap_FS_GetFileList`, `trap_R_RegisterShaderNoMip`, `trap_Print`, `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — all resolved through the VM syscall layer at runtime.

# code/q3_ui/ui_network.c
## File Purpose
Implements the Network Options menu screen within Quake III Arena's legacy UI module (q3_ui). It allows the player to configure their network data rate and navigate between the four System Setup sub-menus (Graphics, Display, Sound, Network).

## Core Responsibilities
- Declare and initialize all menu widgets for the Network Options screen
- Map the `rate` cvar's integer value to a human-readable connection-speed selection
- Write back the selected rate tier to the `rate` cvar on change
- Provide tab-like navigation to sibling option menus (Graphics, Display, Sound)
- Register/cache all required shader assets used by the menu
- Push the constructed menu onto the UI menu stack

## External Dependencies
- `ui_local.h` — pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types, and all `trap_*` syscall declarations
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip`, `color_white`, `color_red`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT`

# code/q3_ui/ui_options.c
## File Purpose
Implements the top-level "System Setup" options menu for Quake III Arena's legacy UI module. It presents four sub-menu navigation buttons (Graphics, Display, Sound, Network) plus a Back button, acting as a hub that dispatches to each specialized settings screen.

## Core Responsibilities
- Initialize and layout the System Setup menu (`optionsmenu_t`) with all UI items
- Pre-cache all artwork (frame bitmaps, back button) used by this menu
- Route activation events to the appropriate sub-menu or pop the menu stack
- Conditionally set fullscreen mode based on whether the client is already connected

## External Dependencies
- **Includes:** `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- **Defined elsewhere:**
  - `Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu` — `ui_qmenu.c` / `ui_atoms.c`
  - `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu` — respective `ui_video.c`, `ui_display.c`, `ui_sound.c`, `ui_network.c`
  - `trap_R_RegisterShaderNoMip`, `trap_GetClientState` — `ui_syscalls.c` (VM syscall wrappers)
  - `color_red`, `color_white` — `ui_atoms.c`

# code/q3_ui/ui_playermodel.c
## File Purpose
Implements the Player Model selection menu in the Quake III Arena q3_ui module. It scans the filesystem for available player model/skin icons, presents them in a paginated 4×4 grid, and persists the selected model/skin to CVars on exit.

## Core Responsibilities
- Build a list of available player models by scanning `models/players/*/icon_*.tga` files
- Render a paginated 4×4 grid of model portrait bitmaps with navigation arrows
- Track the currently selected model/skin, displaying its name and skin name as text
- Render a live 3D player preview using `UI_DrawPlayer` (owner-draw callback)
- Save the selected model to `model`, `headmodel`, `team_model`, and `team_headmodel` CVars
- Handle keyboard navigation (arrow keys, page turning) and mouse clicks on portrait buttons
- Guard 3D player rendering behind a `LOW_MEMORY` (5 MB) threshold

## External Dependencies
- `ui_local.h` — includes `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types, trap syscall declarations
- **Defined elsewhere:** `UI_DrawPlayer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo` (ui_players.c); `Menu_*` functions (ui_qmenu.c); all `trap_*` syscalls (ui_syscalls.c); `uis` global (ui_atoms.c); `menu_move_sound`, `menu_buzz_sound` (ui_qmenu.c)

# code/q3_ui/ui_players.c
## File Purpose
Implements the animated 3D player model preview rendering used in the Q3 UI (e.g., player selection screens). It manages model/skin/weapon loading, animation state machines for legs and torso, and submits all player-related render entities to the renderer each frame.

## Core Responsibilities
- Load and register player model parts (legs, torso, head), skins, and weapon models
- Parse `animation.cfg` files to populate the `animation_t` array
- Drive per-frame animation state machines for legs and torso (sequencing, blending, jump arcs)
- Compute hierarchical bone/tag placement for torso, head, gun, barrel, and muzzle flash entities
- Submit the full multi-part player entity (+ lights, sprite) to the renderer via `trap_R_*` syscalls
- Handle weapon-switch transitions, muzzle flash timing, and barrel spin for machine-gun-style weapons

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`
- **Defined elsewhere:** `bg_itemlist` (game item table), `uis` (global `uiStatic_t`), `weaponChangeSound`, all `trap_*` syscall wrappers, math utilities (`AnglesToAxis`, `MatrixMultiply`, `VectorMA`, etc.), animation constants (`LEGS_JUMP`, `TORSO_ATTACK`, `ANIM_TOGGLEBIT`, `MAX_ANIMATIONS`, etc.)

# code/q3_ui/ui_playersettings.c
## File Purpose
Implements the "Player Settings" menu screen in Quake III Arena's legacy UI module, allowing players to configure their in-game name, handicap level, and effects (rail trail) color, with a live 3D player model preview.

## Core Responsibilities
- Initialize and layout all widgets for the Player Settings menu
- Render custom owner-drawn controls: name field, handicap spinner, effects color picker, and animated player model
- Load current cvar values into UI controls on menu open (`PlayerSettings_SetMenuItems`)
- Persist UI state back to cvars on menu close or navigation (`PlayerSettings_SaveChanges`)
- Handle menu key events, routing escape/mouse2 to save-before-exit
- Preload/cache all required shader assets (`PlayerSettings_Cache`)
- Translate between UI color indices and game color codes via lookup tables

## External Dependencies
- `ui_local.h` — all menu framework types, draw utilities, trap syscalls, `playerInfo_t`, `uiStatic_t uis`
- **Defined elsewhere:** `Menu_AddItem`, `Menu_DefaultKey`, `UI_DrawPlayer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`, `UI_PushMenu`, `UI_PopMenu`, `UI_PlayerModelMenu`, `trap_Cvar_*`, `trap_R_RegisterShaderNoMip`, `trap_Key_GetOverstrikeMode`, `Q_strncpyz`, `Q_CleanStr`, `Q_IsColorString`, `Com_Clamp`, `g_color_table`, `color_white`, `text_color_normal`, `text_color_highlight`

# code/q3_ui/ui_preferences.c
## File Purpose
Implements the "Game Options" preferences menu for the Quake III Arena legacy UI (`q3_ui`). It allows the player to configure gameplay and visual cvars such as crosshair type, dynamic lights, wall marks, team overlay, and file downloading.

## Core Responsibilities
- Declare and initialize all widgets for the Game Options menu screen
- Read current cvar values into widget state on menu open (`Preferences_SetMenuItems`)
- Handle widget activation events and write changed values back to cvars (`Preferences_Event`)
- Provide a custom owner-draw function for the crosshair selector widget (`Crosshair_Draw`)
- Preload all required art assets and crosshair shaders (`Preferences_Cache`)
- Push the constructed menu onto the UI stack as the active screen (`UI_PreferencesMenu`)

## External Dependencies
- **`ui_local.h`** — menu framework types (`menuframework_s`, `menuradiobutton_s`, etc.), `trap_*` syscall wrappers, `UI_Push/PopMenu`, draw utilities
- **`trap_Cvar_VariableValue` / `trap_Cvar_SetValue` / `trap_Cvar_Reset`** — VM syscall layer (defined in `ui_syscalls.c`)
- **`trap_R_RegisterShaderNoMip`** — renderer syscall (defined in `ui_syscalls.c`)
- **`Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu`** — defined in `ui_atoms.c` / `ui_qmenu.c`
- **`Com_Clamp`** — defined in `game/q_shared.c`
- **cvars touched:** `cg_drawCrosshair`, `cg_simpleItems`, `cg_brassTime`, `cg_marks`, `cg_drawCrosshairNames`, `r_dynamiclight`, `r_fastsky`, `r_finish`, `cg_forcemodel`, `cg_drawTeamOverlay`, `cl_allowDownload`, `sv_allowDownload`

# code/q3_ui/ui_qmenu.c
## File Purpose
Implements the core menu framework and all standard widget types for Quake III Arena's legacy UI system (`q3_ui`). It provides initialization, drawing, and input handling for every interactive menu element, plus the top-level menu management routines.

## Core Responsibilities
- Register and cache all shared UI assets (shaders, sounds) via `Menu_Cache`
- Initialize widget bounding boxes and state on `Menu_AddItem`
- Dispatch per-frame drawing for all widget types via `Menu_Draw`
- Route keyboard/mouse input to the focused widget via `Menu_DefaultKey`
- Manage menu cursor movement, focus transitions, and wrap-around via `Menu_AdjustCursor` / `Menu_CursorMoved`
- Provide sound feedback (move, buzz, in/out) for all interactive events
- Support a debug overlay (bounding-box visualization) under `#ifndef NDEBUG`

## External Dependencies
- **`ui_local.h`** — brings in all widget type definitions, flag constants, `uis` global, and `trap_*` syscall declarations.
- **`trap_R_RegisterShaderNoMip`, `trap_R_SetColor`, `trap_S_RegisterSound`, `trap_S_StartLocalSound`** — renderer/audio syscalls, defined in `ui_syscalls.c`.
- **`UI_Draw*`, `UI_FillRect`, `UI_SetColor`, `UI_CursorInRect`** — defined in `ui_atoms.c`.
- **`MenuField_Init`, `MenuField_Draw`, `MenuField_Key`** — defined in `ui_mfield.c`.
- **`UI_PopMenu`** — defined in `ui_atoms.c`.
- **`uis`** (`uiStatic_t`) — singleton global defined in `ui_atoms.c`.
- **`Menu_ItemAtCursor`** — defined in this file; also declared `extern` in `ui_local.h` for use by other modules.

# code/q3_ui/ui_rankings.c
## File Purpose
Implements the in-game "Rankings" overlay menu for Quake III Arena's online ranking system (GRank). It presents context-sensitive options (login, logout, sign up, spectate, setup, leave arena) based on the player's current ranking status.

## Core Responsibilities
- Initialize and display the rankings popup menu with a decorative frame
- Show/hide/gray out menu items dynamically based on `client_status` cvar (grank status)
- Route menu events to appropriate UI screens or game commands
- Provide custom field draw helpers for name and password input fields (used by login/signup menus)
- Pre-cache the frame shader asset

## External Dependencies
- `ui_local.h` — pulls in all menu framework types, trap syscalls, color tables, and UI helper declarations
- **Defined elsewhere:** `grank_status_t`, `QGR_STATUS_*` constants (GRank headers), `trap_CL_UI_RankUserRequestLogout`, `UI_LoginMenu`, `UI_SignupMenu`, `UI_SetupMenu`, `UI_ForceMenuOff`, `UI_PushMenu`, `Menu_AddItem`, `UI_DrawChar`, `trap_Key_GetOverstrikeMode`, `Q_CleanStr`, `Q_strncpyz`, `g_color_table`, `ColorIndex`, `color_white`, `text_color_normal`, `text_color_highlight`

# code/q3_ui/ui_rankstatus.c
## File Purpose
Implements a modal status dialog for the GRank (Global Ranking) online ranking system, displaying error or result messages when a ranking operation completes. It maps `grank_status_t` codes to human-readable strings and routes the user to appropriate follow-up menus on dismissal.

## Core Responsibilities
- Read `client_status` cvar to determine the current `grank_status_t` code
- Map ranking status codes to display strings (e.g., "Invalid password", "Timed out")
- Build and display a simple two-item menu: a static message and an OK button
- On OK, pop this menu and push the appropriate follow-up menu (rankings, login, signup) based on the original status code
- Early-exit silently for benign statuses (`QGR_STATUS_NEW`, `QGR_STATUS_PENDING`, `QGR_STATUS_SPECTATOR`, `QGR_STATUS_ACTIVE`)
- Pre-cache the frame shader via `RankStatus_Cache`

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, menu framework types, all `trap_*` syscall declarations)
- **Defined elsewhere:**
  - `grank_status_t` and its `QGR_STATUS_*` constants — ranking system types (defined in ranking headers pulled through `ui_local.h`)
  - `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip`, `trap_CL_UI_RankUserReset` — VM syscall stubs (`ui_syscalls.c`)
  - `UI_PopMenu`, `UI_PushMenu`, `UI_ForceMenuOff` — menu stack management (`ui_atoms.c`)
  - `UI_RankingsMenu`, `UI_LoginMenu`, `UI_SignupMenu` — sibling ranking UI screens
  - `Menu_AddItem` — menu framework (`ui_qmenu.c`)
  - `colorRed` — shared color constant (`ui_qmenu.c` / `q_shared.c`)

# code/q3_ui/ui_removebots.c
## File Purpose
Implements the in-game "Remove Bots" menu for Quake III Arena's legacy UI module. It allows a human player to view currently connected bot clients and kick one by client number via a console command.

## Core Responsibilities
- Enumerate active bot clients from server config strings by checking for a non-zero `skill` field
- Display up to 7 bot names in a scrollable list
- Track which bot entry is selected and visually distinguish it (orange vs. white color)
- Issue a `clientkick <num>` command when the user activates the Delete button
- Register and cache all required artwork shaders on demand
- Push/pop the menu onto the UI menu stack

## External Dependencies
- `ui_local.h` — all menu framework types, widget types, trap syscall declarations, color vectors
- `trap_GetConfigString` — reads `CS_SERVERINFO` and `CS_PLAYERS + n` (defined in engine/syscall layer)
- `trap_Cmd_ExecuteText` — issues console commands to the engine
- `trap_R_RegisterShaderNoMip` — registers 2D art assets
- `Info_ValueForKey`, `Q_strncpyz`, `Q_CleanStr` — defined in `q_shared.c`
- `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — defined in `ui_atoms.c` / `ui_qmenu.c`
- `MAX_BOTS` — defined in `bg_public.h`

# code/q3_ui/ui_saveconfig.c
## File Purpose
Implements the "Save Config" menu screen for Quake III Arena's legacy UI module (`q3_ui`). It presents a full-screen dialog allowing the player to type a filename and write the current game configuration to a `.cfg` file via a console command.

## Core Responsibilities
- Initialize and layout the Save Config menu widgets (banner, background, text field, back/save buttons)
- Pre-cache all bitmap art assets used by the menu
- Handle the "Back" button event by popping the menu stack
- Handle the "Save" button event by stripping the file extension and dispatching a `writeconfig` command
- Provide a custom owner-draw callback for the filename input field
- Expose the menu entry point (`UI_SaveConfigMenu`) and asset cache function to the rest of the UI module

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_ItemAtCursor` — `ui_qmenu.c` / `ui_atoms.c`
  - `MField_Draw` — `ui_mfield.c`
  - `UI_DrawProportionalString`, `UI_FillRect` — `ui_atoms.c`
  - `trap_R_RegisterShaderNoMip`, `trap_Cmd_ExecuteText` — `ui_syscalls.c` (VM syscall wrappers)
  - `COM_StripExtension` — `q_shared.c`
  - `va` — `q_shared.c`
  - Color constants (`color_orange`, `colorBlack`, `colorRed`, `text_color_highlight`) — `ui_qmenu.c`

# code/q3_ui/ui_serverinfo.c
## File Purpose
Implements the "Server Info" UI menu in Quake III Arena's legacy q3_ui module. It displays key-value pairs from the current server's config string and provides "Add to Favorites" and "Back" actions.

## Core Responsibilities
- Fetch and display the server's `CS_SERVERINFO` config string as a key-value table
- Vertically center the info table based on the number of lines
- Allow the player to add the current server to the favorites list (cvars `server1`–`serverN`)
- Prevent the "Add to Favorites" action when a local server is running (`sv_running`)
- Pre-cache UI art assets via `trap_R_RegisterShaderNoMip`
- Provide keyboard and mouse event routing through the standard menu framework

## External Dependencies
- `ui_local.h` — menu framework types, trap functions, draw utilities, `MAX_FAVORITESERVERS`
- `trap_GetConfigString` / `CS_SERVERINFO` — defined in engine/qcommon layer
- `Info_NextPair` — defined in `q_shared.c`
- `UI_PushMenu`, `UI_PopMenu`, `Menu_Draw`, `Menu_DefaultKey`, `Menu_AddItem` — defined in `ui_qmenu.c`
- `UI_DrawString` — defined in `ui_atoms.c`
- `trap_R_RegisterShaderNoMip`, `trap_Cvar_*` — syscall stubs defined in `ui_syscalls.c`

# code/q3_ui/ui_servers2.c
## File Purpose
Implements the Quake III Arena multiplayer server browser menu ("Arena Servers"), handling server discovery, ping querying, filtering, sorting, and connection initiation. It manages four server source types: Local, Internet (Global), MPlayer, and Favorites.

## Core Responsibilities
- Initialize and render the server browser menu with all UI controls
- Manage ping request queues to discover and measure server latency
- Filter server list by game type, full/empty status, and max ping
- Sort server list by hostname, map, open slots, game type, or ping
- Persist and load favorite server addresses via cvars (`server1`–`server16`)
- Handle PunkBuster enable/disable confirmation dialogs
- Connect to a selected server via `connect` command

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h`, all menu framework types and trap syscalls
- **Defined elsewhere:** `trap_LAN_*` (server list and ping syscalls), `trap_Cmd_ExecuteText`, `trap_Cvar_*`, `trap_R_RegisterShaderNoMip`, `Menu_Draw`, `Menu_AddItem`, `Menu_DefaultKey`, `ScrollList_Key`, `UI_PushMenu`, `UI_PopMenu`, `UI_ConfirmMenu_Style`, `UI_SpecifyServerMenu`, `UI_StartServerMenu`, `UI_Message`, `uis` (global UI state), `qsort` (libc)

# code/q3_ui/ui_setup.c
## File Purpose
Implements the Setup menu screen for Quake III Arena's legacy UI module (`q3_ui`). It presents a top-level configuration hub from which players navigate to sub-menus covering player settings, controls, graphics, game options, and CD key entry.

## Core Responsibilities
- Define and initialize all menu items for the Setup screen layout
- Route menu item activation events to their respective sub-menu functions
- Conditionally show the "DEFAULTS" option only when not in-game (i.e., `cl_paused == 0`)
- Confirm and execute a full configuration reset via `exec default.cfg` / `cvar_restart` / `vid_restart`
- Pre-cache all bitmap assets used by the Setup screen
- Expose public entry points (`UI_SetupMenu`, `UI_SetupMenu_Cache`) for the broader UI system

## External Dependencies
- **Includes:** `ui_local.h` (aggregates `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework declarations, and all trap syscall prototypes)
- **Defined elsewhere:**
  - `UI_PlayerSettingsMenu`, `UI_ControlsMenu`, `UI_GraphicsOptionsMenu`, `UI_PreferencesMenu`, `UI_CDKeyMenu` — sub-menu entry points in their respective `.c` files
  - `UI_ConfirmMenu` — `ui_confirm.c`
  - `UI_PushMenu`, `UI_PopMenu` — `ui_atoms.c`
  - `Menu_AddItem`, `Menu_Draw` — `ui_qmenu.c`
  - `trap_*` syscalls — `ui_syscalls.c` (VM trap layer)
  - `color_white`, `color_red`, `color_yellow` — `ui_atoms.c`

# code/q3_ui/ui_signup.c
## File Purpose
Implements the user account sign-up menu for Quake III Arena's GRank (Global Rankings) online ranking system. It provides a form UI for new players to register a ranked account by supplying a name, password (with confirmation), and email address.

## Core Responsibilities
- Define and initialize all UI widgets for the sign-up form (labels, input fields, buttons)
- Validate that the password and confirmation fields match before submission
- Invoke `trap_CL_UI_RankUserCreate` to submit registration data to the rankings backend
- Conditionally disable all input fields if the player's `client_status` indicates they are not eligible to sign up (i.e., already registered)
- Preload the frame bitmap asset via `Signup_Cache`
- Push the initialized menu onto the UI menu stack via `UI_SignupMenu`

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types, and all trap declarations)
- **Defined elsewhere:**
  - `trap_CL_UI_RankUserCreate` — ranking system syscall, not declared in the bundled header (GRank-specific extension)
  - `Rankings_DrawName`, `Rankings_DrawPassword`, `Rankings_DrawText` — ownerdraw callbacks defined in `ui_rankings.c`
  - `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` — defined in `ui_atoms.c`
  - `grank_status_t`, `QGR_STATUS_NEW`, `QGR_STATUS_SPECTATOR` — defined in GRank headers (not shown)
  - `Menu_AddItem` — defined in `ui_qmenu.c`

# code/q3_ui/ui_sound.c
## File Purpose
Implements the Sound Options menu screen within Quake III Arena's legacy UI module (`q3_ui`). It allows players to configure effects volume, music volume, and sound quality (sample rate/compression) through a standard menu framework.

## Core Responsibilities
- Initialize and lay out all sound options menu widgets (sliders, spin control, navigation tabs, decorative bitmaps)
- Read current sound CVars (`s_volume`, `s_musicvolume`, `s_compression`) to populate widget state on open
- Write CVar changes back to the engine when the user adjusts controls
- Navigate to sibling option menus (Graphics, Display, Network) or go back
- Trigger `snd_restart` when sound quality is changed, requiring a sound system reload

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types and prototypes)
- **Defined elsewhere:**
  - `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip` — syscall wrappers in `ui_syscalls.c`
  - `UI_PopMenu`, `UI_PushMenu`, `UI_ForceMenuOff`, `Menu_AddItem`, `Menu_SetCursorToItem` — menu framework in `ui_atoms.c` / `ui_qmenu.c`
  - `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_NetworkOptionsMenu` — sibling option menu files
  - `color_white`, `color_red`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT` — shared UI constants

# code/q3_ui/ui_sparena.c
## File Purpose
Handles the launch sequence for a single-player arena in Quake III Arena's UI layer. It configures the necessary CVars and issues the server command to start a specific SP map.

## Core Responsibilities
- Ensures `sv_maxclients` is at least 8 before starting an SP arena
- Resolves the numeric SP level index from arena metadata, with special-case handling for "training" and "final" arenas
- Writes the resolved level selection into the `ui_spSelection` CVar for downstream use
- Executes the `spmap` command to load the chosen map

## External Dependencies
- **Includes:** `ui_local.h` (which pulls in `q_shared.h`, `bg_public.h`, trap syscall declarations)
- **Defined elsewhere:**
  - `trap_Cvar_VariableValue`, `trap_Cvar_SetValue`, `trap_Cmd_ExecuteText` — UI syscall stubs (`ui_syscalls.c`)
  - `Info_ValueForKey`, `Q_stricmp`, `atoi`, `va` — shared utilities (`q_shared.c`)
  - `UI_GetNumSPTiers`, `ARENAS_PER_TIER` — SP game info module (`ui_gameinfo.c` / `bg_public.h`)

# code/q3_ui/ui_specifyleague.c
## File Purpose
Implements the "Specify League" UI menu for Quake III Arena's Global Rankings system, allowing players to enter a username, query available leagues for that player, and select one to set as the active `sv_leagueName` cvar.

## Core Responsibilities
- Initialize and lay out the Specify League menu screen with decorative bitmaps, a player name text field, a scrollable league list, and navigation buttons
- Query the Global Rankings backend for leagues associated with a given player name via `trap_CL_UI_RankGetLeauges`
- Populate a fixed-size list box with league names retrieved from numbered cvars (`leaguename1`, `leaguename2`, …)
- Re-query the league list when the player name field loses focus and the name has changed
- Write the selected league name to `sv_leagueName` cvar on back/confirm
- Pre-cache all required UI art shaders via `SpecifyLeague_Cache`

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- `trap_CL_UI_RankGetLeauges` — Global Rankings syscall, defined in `ui_syscalls.c` / engine; not declared in the bundled header (likely a GRank extension)
- `trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer`, `trap_R_RegisterShaderNoMip` — engine syscalls declared in `ui_local.h`
- `Menu_AddItem`, `ScrollList_Key`, `UI_PushMenu`, `UI_PopMenu` — defined in `ui_qmenu.c` / `ui_atoms.c`
- `Q_strncpyz`, `Q_strncmp`, `va` — string utilities from `q_shared.c`

# code/q3_ui/ui_specifyserver.c
## File Purpose
Implements the "Specify Server" UI menu, allowing players to manually enter a server IP address and port number to connect to directly. It is a simple two-field input form within the Q3 legacy UI module.

## Core Responsibilities
- Define and initialize all menu items (banner, decorative frames, address/port fields, go/back buttons)
- Handle user activation events for "Go" (connect) and "Back" (pop menu) buttons
- Preload/cache all required bitmap art assets via the renderer
- Build and dispatch the `connect <address>:<port>` command string to the engine
- Push the assembled menu onto the active UI menu stack

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types/macros)
- **Defined elsewhere:** `Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip`, `Com_sprintf`, `va`, `color_white` — all provided by the broader Q3 UI/engine runtime

# code/q3_ui/ui_splevel.c
## File Purpose
Implements the single-player level selection menu for Quake III Arena, allowing players to browse tier-based arena sets, select maps, view completion status, and navigate to the skill selection screen.

## Core Responsibilities
- Initialize and layout the level select menu with up to 4 level thumbnail bitmaps per tier
- Handle tier navigation via left/right arrow buttons
- Display player icon, awards/medals, and bot opponent portraits
- Track and display level completion status with skill-rated completion images
- Handle special-case tiers (training and final) with single-map display
- Provide reset-game confirmation flow and custom/skirmish navigation

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `tr_types.h`, `ui_public.h`
- **Defined elsewhere:** `UI_GetArenaInfoByNumber`, `UI_GetSpecialArenaInfo`, `UI_GetBotInfoByName`, `UI_GetBestScore`, `UI_GetAwardLevel`, `UI_GetCurrentGame`, `UI_NewGame`, `UI_GetNumSPTiers`, `UI_GetNumSPArenas`, `UI_SPSkillMenu`, `UI_StartServerMenu`, `UI_PlayerSettingsMenu`, `UI_ConfirmMenu`, `Menu_Draw`, `Menu_AddItem`, `Bitmap_Init`, `Menu_SetCursorToItem`, `UI_PushMenu`, `UI_PopMenu`, all `trap_*` syscalls, `uis` global, `ui_medalPicNames[]`, `ui_medalSounds[]`, `ARENAS_PER_TIER`, `PULSE_DIVISOR`

# code/q3_ui/ui_sppostgame.c
## File Purpose
Implements the single-player postgame menu for Quake III Arena, displayed after a match ends. It orchestrates a three-phase animated sequence: podium presentation, award medal display, then interactive buttons for replay/next/menu navigation.

## Core Responsibilities
- Parse postgame command arguments (scores, ranks, award stats) into menu state
- Drive a three-phase timed presentation (podium → awards → navigation buttons)
- Display and animate per-award medals with sounds
- Evaluate tier/level progression logic to determine the "Next" level destination
- Trigger tier cinematic videos upon tier completion
- Persist best scores and award data via `UI_SetBestScore` / `UI_LogAwardData`
- Register and play winner/loser music and announcement sounds

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, `keycodes.h`, all menu/trap declarations
- **Defined elsewhere:** `UI_GetArenaInfoByMap`, `UI_GetArenaInfoByNumber`, `UI_TierCompleted`, `UI_ShowTierVideo`, `UI_GetNumSPTiers`, `UI_GetSpecialArenaInfo`, `UI_SetBestScore`, `UI_LogAwardData`, `UI_GetAwardLevel`, `UI_GetCurrentGame`, `UI_SPArena_Start`, `Menu_*` functions, all `trap_*` syscalls, `uis` global, draw utilities (`UI_DrawProportionalString`, `UI_DrawNamedPic`, `UI_DrawString`)

# code/q3_ui/ui_spreset.c
## File Purpose
Implements the single-player "Reset Game" confirmation dialog for Quake III Arena's UI module. It presents a YES/NO prompt to the player and, on confirmation, wipes all single-player progress data and restarts the level menu from the beginning.

## Core Responsibilities
- Renders the reset confirmation dialog with a decorative frame and warning text
- Handles YES/NO menu item selection via mouse and keyboard (including `Y`/`N` hotkeys)
- On confirmation: calls `UI_NewGame()`, resets `ui_spSelection` to 0, pops the current menu stack entries, and re-launches the SP level menu
- Caches the background frame shader on demand
- Positions the `YES / NO` text layout dynamically using proportional string width calculations
- Sets fullscreen vs. overlay mode based on whether a game session is currently connected

## External Dependencies
- `ui_local.h` — pulls in all UI framework types, menu item types, trap syscalls, draw utilities, and SP game info functions
- **Defined elsewhere:** `UI_NewGame` (`ui_gameinfo.c`), `UI_SPLevelMenu` (`ui_spLevel.c`), `UI_PopMenu` / `UI_PushMenu` / `UI_DrawNamedPic` / `UI_DrawProportionalString` / `UI_ProportionalStringWidth` (`ui_atoms.c`), `Menu_Draw` / `Menu_DefaultKey` / `Menu_AddItem` / `Menu_SetCursorToItem` (`ui_qmenu.c`), `trap_*` syscall wrappers (`ui_syscalls.c`), `trap_R_RegisterShaderNoMip` (renderer via VM syscall)

# code/q3_ui/ui_spskill.c
## File Purpose
Implements the single-player difficulty selection menu in Quake III Arena's UI module. It presents five skill levels ("I Can Win" through "NIGHTMARE!"), persists the selection to the `g_spSkill` cvar, and transitions into the arena start flow.

## Core Responsibilities
- Initialize and lay out all menu widgets for the skill selection screen
- Highlight the currently selected skill in white; all others in red
- Update `g_spSkill` cvar when the player selects a difficulty
- Swap the displayed skill-level illustration (`art_skillPic`) on selection change
- Play a special sound for NIGHTMARE difficulty; silence sound otherwise
- Navigate back to the previous menu or forward to `UI_SPArena_Start`
- Pre-cache all shaders and sounds required by this menu

## External Dependencies
- `ui_local.h` — menu framework types, widget types, trap syscall declarations, helper functions
- **Defined elsewhere:** `UI_SPArena_Start`, `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `Menu_DefaultKey`, `Com_Clamp`, all `trap_*` syscall wrappers, `color_red`, `color_white`

# code/q3_ui/ui_startserver.c
## File Purpose
Implements three interconnected UI menus for launching a multiplayer or single-player server: the **Start Server** map-selection menu, the **Server Options** configuration menu, and the **Bot Select** picker menu. Together they form a wizard-style flow: pick a map → configure options/bots → execute the server launch.

## Core Responsibilities
- Display a paginated grid of level-shot thumbnails filtered by game type for map selection
- Allow game type selection (FFA, Team DM, Tournament, CTF) and re-filter the map list accordingly
- Provide server configuration controls: frag/time/capture limits, friendly fire, pure server, dedicated mode, hostname, bot skill, PunkBuster
- Manage up to 12 player slots as Open/Bot/Closed with optional team assignment
- Display a paginated bot portrait grid for bot selection, sorted alphabetically
- Build and execute the `map` command along with `addbot` and `team` commands to start the server

## External Dependencies
- **Includes:** `ui_local.h` (menu framework, trap syscalls, shared types)
- **Defined elsewhere:** `punkbuster_items[]` (extern from `ui_servers2.c`); `UI_ServerOptionsMenu` forward-declared static but called from `StartServer_MenuEvent`
- **Trap syscalls used:** `trap_R_RegisterShaderNoMip`, `trap_Cvar_SetValue`, `trap_Cvar_Set`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cmd_ExecuteText`
- **UI info queries:** `UI_GetNumArenas`, `UI_GetArenaInfoByNumber`, `UI_GetArenaInfoByMap`, `UI_GetNumBots`, `UI_GetBotInfoByNumber`, `UI_GetBotInfoByName`, `Info_ValueForKey`

# code/q3_ui/ui_team.c
## File Purpose
Implements the in-game Team Selection overlay menu for Quake III Arena, allowing players to join the red team, blue team, free-for-all, or spectate. Menu items are conditionally grayed out based on the current server game type.

## Core Responsibilities
- Define and initialize the team selection menu (`s_teammain`)
- Register the decorative frame shader asset via cache call
- Handle menu item activation events by sending server commands (`cmd team red/blue/free/spectator`)
- Query `CS_SERVERINFO` to determine current game type and disable irrelevant options
- Push the initialized menu onto the UI menu stack

## External Dependencies
- `ui_local.h` — pulls in `menuframework_s`, `menubitmap_s`, `menutext_s`, `menucommon_s`, `QM_ACTIVATED`, `QMF_*` flags, `MTYPE_*` constants, game type enums (`GT_TEAM`, `GT_CTF`, etc.), `CS_SERVERINFO`, and all `trap_*` / `UI_*` function declarations
- `trap_Cmd_ExecuteText` — defined in `ui_syscalls.c`, bridges to engine
- `trap_GetConfigString` — defined in `ui_syscalls.c`, bridges to engine
- `trap_R_RegisterShaderNoMip` — defined in `ui_syscalls.c`, bridges to renderer
- `Info_ValueForKey` — defined in `q_shared.c`
- `UI_ForceMenuOff`, `UI_PushMenu`, `Menu_AddItem` — defined in `ui_atoms.c` / `ui_qmenu.c`

# code/q3_ui/ui_teamorders.c
## File Purpose
Implements the in-game Team Orders menu for Quake III Arena, allowing players to issue commands to bot teammates. It presents a two-step selection UI: first choose a bot (or "Everyone"), then choose an order, which is transmitted as a `say_team` chat message.

## Core Responsibilities
- Build a dynamic list of bot teammates from server config strings
- Render a scrollable, owner-drawn proportional-font list widget
- Handle two-phase selection: bot target → order message
- Format and dispatch `say_team` commands with the selected bot name interpolated
- Guard menu access (team game only, non-spectators)
- Pre-cache required artwork shaders

## External Dependencies
- **Includes:** `ui_local.h` → pulls in `q_shared.h`, `bg_public.h`, UI framework types and trap declarations
- **Defined elsewhere:** `trap_GetClientState`, `trap_GetConfigString`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip`, `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem`, `Menu_ItemAtCursor`, `Menu_DefaultKey`, `UI_DrawProportionalString`, `UI_CursorInRect`, `Com_sprintf`, `va`, `Q_strncpyz`, `Q_CleanStr`, `Info_ValueForKey`, `uis` (global UI state), game constants `GT_CTF`, `GT_TEAM`, `TEAM_SPECTATOR`, `CS_SERVERINFO`, `CS_PLAYERS`

# code/q3_ui/ui_video.c
## File Purpose
Implements two UI menus for Quake III Arena: the **Driver Info** screen (read-only display of OpenGL vendor/renderer/extension strings) and the **Graphics Options** screen (interactive controls for video settings such as resolution, color depth, texture quality, and geometry detail).

## Core Responsibilities
- Build and display the Driver Info menu, parsing and rendering GL extension strings in two columns
- Build and display the Graphics Options menu with spin controls, sliders, and bitmaps for all major renderer cvars
- Apply pending video changes by writing renderer cvars and issuing `vid_restart`
- Track initial video state (`s_ivo`) to determine when the "Apply" button should be shown
- Match current settings against predefined quality presets (High/Normal/Fast/Fastest/Custom)
- Navigate between sibling option menus (Display, Sound, Network) via tab-style text buttons
- Preload all UI art shaders via cache functions

## External Dependencies
- `ui_local.h` — menu framework types, `uis` global (`uiStatic_t`), all `trap_*` syscalls, `UI_Push/PopMenu`, `UI_DrawString`, color constants
- `uis.glconfig` (`glconfig_t`) — GL vendor/renderer/version/extensions strings, driver type, hardware type, color/depth/stencil bits
- External menu functions: `Menu_Draw`, `Menu_AddItem`, `Menu_SetCursorToItem`
- External navigation targets (defined elsewhere): `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`
- Renderer cvars written: `r_mode`, `r_fullscreen`, `r_colorbits`, `r_depthbits`, `r_stencilbits`, `r_texturebits`, `r_picmip`, `r_vertexLight`, `r_lodBias`, `r_subdivisions`, `r_textureMode`, `r_allowExtensions`, `r_glDriver`
- `OPENGL_DRIVER_NAME`, `_3DFX_DRIVER_NAME` — defined elsewhere (platform headers)

# code/qcommon/cm_load.c
## File Purpose
Loads and parses a Quake III BSP map file into the collision map (`clipMap_t cm`) used by the collision detection system. It deserializes all BSP lumps into runtime structures and initializes the box hull and area flood connectivity after loading.

## Core Responsibilities
- Read and validate a BSP file from disk (or BSPC tool path)
- Deserialize each BSP lump (shaders, planes, nodes, leafs, brushes, brush sides, submodels, visibility, patches, entity string) into hunk-allocated runtime structures
- Endian-swap all numeric fields from little-endian BSP format to host byte order
- Compute and expose a checksum for map integrity verification
- Set up the synthetic box hull for AABB-as-brush-model queries
- Provide accessor functions for cluster/area/entity string data

## External Dependencies
- `cm_local.h` → `q_shared.h`, `qcommon.h`, `cm_polylib.h` — shared types and engine utilities
- `bspc/l_qfiles.h` — BSPC tool file abstraction (included only when `BSPC` is defined)
- **Defined elsewhere:** `Hunk_Alloc`, `Com_Memcpy`, `Com_Memset`, `Com_Error`, `Com_DPrintf`, `Com_BlockChecksum`, `FS_ReadFile`, `FS_FreeFile`, `Cvar_Get`, `LittleLong`, `LittleFloat`, `PlaneTypeForNormal`, `SetPlaneSignbits`, `VectorCopy`, `VectorClear`, `Q_strncpyz`, `CM_GeneratePatchCollide`, `CM_ClearLevelPatches`, `CM_FloodAreaConnections`

# code/qcommon/cm_local.h
## File Purpose
Private header for the collision map (CM) subsystem, defining all internal data structures, the global clip map state, and declaring the internal API shared across `cm_load.c`, `cm_test.c`, `cm_trace.c`, and `cm_patch.c`. It is never included by code outside the `qcommon/cm_*` family.

## Core Responsibilities
- Define the in-memory BSP collision tree types (`cNode_t`, `cLeaf_t`, `cmodel_t`, `cbrush_t`, `cPatch_t`, `cArea_t`)
- Define the monolithic `clipMap_t` structure that holds the entire loaded collision world
- Declare the global `cm` instance and all CM-related debug/trace counters
- Define the per-trace working state (`traceWork_t`, `sphere_t`) used during box/capsule sweeps
- Declare the leaf-enumeration utility type (`leafList_t`) and its callbacks
- Declare internal cross-file functions for box queries, leaf enumeration, and patch collision

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `cplane_t`, `trace_t`, `clipHandle_t`, `qboolean`, `dshader_t`
- `qcommon.h` → `cm_public.h` — public CM API types
- `cm_polylib.h` — `winding_t` (used only by debug visualization in `cm_debug`)
- `patchCollide_s` — defined in `cm_patch.c` (forward-declared here as an incomplete struct)
- `dshader_t` — defined in `qfiles.h` (BSP on-disk shader lump entry)

# code/qcommon/cm_patch.c
## File Purpose
Implements collision detection for quadratic Bezier patch meshes (curved surfaces) in Quake III Arena. It converts a patch control point grid into a flat facet/plane representation, then provides trace and position-test entry points used by the collision model system.

## Core Responsibilities
- Subdivide a quadratic Bezier grid until all curve segments are within `SUBDIVIDE_DISTANCE` of linear
- Remove degenerate (duplicate) columns and transpose the grid to subdivide both axes
- Build a plane list and facet list (`patchCollide_t`) from the subdivided grid triangles
- Add axial and edge bevel planes to each facet to prevent tunneling
- Perform swept-volume and point traces against a `patchCollide_t`
- Perform static position (overlap) tests against a `patchCollide_t`
- Render debug geometry for patch collision surfaces via a callback

## External Dependencies
- `cm_local.h` → `q_shared.h`, `qcommon.h`, `cm_polylib.h` (winding utilities: `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `FreeWinding`, `CopyWinding`)
- `cm_patch.h` — type and constant definitions (`patchPlane_t`, `facet_t`, `patchCollide_t`, `cGrid_t`, `MAX_FACETS`, `MAX_PATCH_PLANES`, `MAX_GRID_SIZE`, `SUBDIVIDE_DISTANCE`, etc.)
- **Defined elsewhere:** `Hunk_Alloc`, `Com_Error`, `Com_Printf`, `Com_DPrintf`, `Com_Memset`, `Com_Memcpy`, `Cvar_Get`, `VectorMA`, `DotProduct`, `CrossProduct`, `VectorNormalize`, `VectorNegate`, `VectorSubtract`, `VectorAdd`, `VectorCopy`, `VectorClear`, `Vector4Copy`, `AddPointToBounds`, `ClearBounds`, `cm_playerCurveClip`, `BotDrawDebugPolygons`

# code/qcommon/cm_patch.h
## File Purpose
Defines the data structures and entry points for Quake III Arena's curved-surface (patch mesh) collision system. It bridges the patch tessellation pipeline with the broader collision model (`cm_`) subsystem.

## Core Responsibilities
- Declare types for patch collision geometry (`patchPlane_t`, `facet_t`, `patchCollide_t`, `cGrid_t`)
- Define capacity limits for facets and planes used during patch collision generation
- Expose the public entry point `CM_GeneratePatchCollide` to callers in the collision module
- Document known issues and design tradeoffs for curved-surface collision (via header comments)

## External Dependencies
- **Includes (implicit):** Relies on `cm_local.h` (or `qcommon.h`) for `vec3_t`, `qboolean`, `traceWork_t`.
- **Defined elsewhere:** `CM_ClearLevelPatches`, `CM_TraceThroughPatchCollide`, `CM_PositionTestInPatchCollide`, `CM_DrawDebugSurface` — all implemented in `cm_patch.c`. `traceWork_t` defined in `cm_local.h`.

# code/qcommon/cm_polylib.c
## File Purpose
Provides polygon (winding) geometry utilities used exclusively by the collision map (`cm_`) debug and visualization tools. It is not part of the runtime collision pipeline itself, only supporting diagnostic/visualization code.

## Core Responsibilities
- Allocate and free `winding_t` polygon objects with tracking counters
- Compute geometric properties: plane, area, bounds, center
- Clip windings against planes (both destructive and non-destructive variants)
- Validate winding convexity and planarity
- Classify winding position relative to a plane
- Construct a convex hull from multiple coplanar windings

## External Dependencies
- `cm_local.h` → transitively pulls in `q_shared.h`, `qcommon.h`, `cm_polylib.h`
- `Z_Malloc` / `Z_Free` — zone allocator (defined in `common.c`)
- `Com_Memset`, `Com_Memcpy`, `Com_Error` — common utilities (defined in `common.c`)
- `DotProduct`, `CrossProduct`, `VectorSubtract`, `VectorNormalize2`, `VectorMA`, `VectorScale`, `VectorLength` — math macros/functions from `q_shared.h`
- `SIDE_FRONT`, `SIDE_BACK`, `SIDE_ON`, `SIDE_CROSS`, `ON_EPSILON`, `MAX_MAP_BOUNDS`, `MAX_POINTS_ON_WINDING` — constants defined elsewhere (likely `cm_polylib.h` / `q_shared.h`)

# code/qcommon/cm_polylib.h
## File Purpose
Declares the `winding_t` polygon type and its associated operations for convex polygon manipulation. Used exclusively by the collision model debug/visualization subsystem (`cm_` functions), not by general gameplay or rendering.

## Core Responsibilities
- Define the `winding_t` convex polygon primitive
- Declare allocation, copy, and deallocation routines for windings
- Declare geometric query operations (area, center, bounds, plane, side classification)
- Declare clipping and chopping operations against planes
- Declare convex hull construction helper
- Define plane-side and clipping constants

## External Dependencies
- `vec3_t`, `vec_t` — defined in `q_shared.h` / `qcommon.h`
- All function bodies defined in `code/qcommon/cm_polylib.c`
- `MAX_POINTS_ON_WINDING`, `ON_EPSILON`, `CLIP_EPSILON` constants are self-contained in this header

# code/qcommon/cm_public.h
## File Purpose
Public API header for Quake III Arena's collision map (CM) subsystem. It declares all externally-visible functions that other engine modules (server, client, game) use to query the BSP collision world: map loading, spatial queries, trace/sweep tests, PVS lookups, and area portal management.

## Core Responsibilities
- Declare the map load/unload lifecycle interface (`CM_LoadMap`, `CM_ClearMap`)
- Expose clip handle acquisition for inline BSP models and temporary box models
- Provide point and box content queries (brush contents flags)
- Declare box/capsule trace (sweep) functions against the collision world
- Export PVS (Potential Visibility Set) cluster queries
- Expose leaf/area/portal connectivity queries
- Declare tag interpolation, mark fragment, and debug surface utilities

## External Dependencies
- `qfiles.h` — BSP file format structs (`dheader_t`, `dleaf_t`, etc.) and limits; `vmHeader_t`, model format structs
- `clipHandle_t`, `trace_t`, `vec3_t`, `orientation_t`, `markFragment_t`, `qboolean`, `byte` — all defined in `q_shared.h` or `cm_local.h`, included transitively by consumers
- Implementation bodies defined across `cm_load.c`, `cm_trace.c`, `cm_test.c`, `cm_patch.c`, `cm_tag.c`, `cm_marks.c`

# code/qcommon/cm_test.c
## File Purpose
Implements point/area spatial queries and area connectivity (PVS/portals) for Quake III Arena's collision map system. It provides BSP tree traversal to locate leafs and brush contents at a point, as well as flood-fill logic for area portal connectivity used in entity culling.

## Core Responsibilities
- Traverse the BSP tree to find which leaf a point occupies
- Query content flags (solid, water, etc.) at a world-space point
- Enumerate all leafs or brushes touching an AABB
- Expose PVS (Potentially Visible Set) cluster data per cluster index
- Maintain and recompute area portal flood-connectivity
- Provide a bit-vector of areas reachable from a given area for snapshot culling

## External Dependencies
- `cm_local.h` → pulls in `q_shared.h`, `qcommon.h`, `cm_polylib.h`
- **Defined elsewhere:** `cm` (clipMap_t global, loaded by `cm_load.c`), `CM_ClipHandleToModel` (`cm_load.c`), `BoxOnPlaneSide` (`q_shared.c`/math), `AngleVectors`, `DotProduct`, `VectorCopy`, `VectorSubtract` (math macros/functions), `Com_Error`, `Com_Memset` (qcommon), `cm_noAreas` CVar (registered in `cm_main.c` or similar).

# code/qcommon/cm_trace.c
## File Purpose
Implements all collision trace and position-test logic for Quake III Arena's clip-map system. It sweeps axis-aligned bounding boxes (AABB), oriented capsules, and points through BSP trees and patch surfaces, returning the first solid intersection fraction and contact plane.

## Core Responsibilities
- Point/AABB/capsule position overlap tests against brushes, patches, and the BSP tree
- Swept-volume trace (AABB and capsule) through brushes and patch collide surfaces
- Capsule-vs-capsule and AABB-vs-capsule collision dispatch
- BSP tree traversal routing swept traces to the correct leaf nodes
- Coordinate transformation (rotation/translation) for traces against rotated sub-models
- Per-trace `traceWork_t` setup: symmetric sizing, signbit corner offsets, bounds, sphere params

## External Dependencies
- **`cm_local.h`** — all type definitions (`traceWork_t`, `cbrush_t`, `cLeaf_t`, `clipMap_t`, `sphere_t`, etc.), extern declarations, and `SURFACE_CLIP_EPSILON`
- **`q_shared.h`** (via `cm_local.h`) — `vec3_t`, `trace_t`, `cplane_t`, `qboolean`, `VectorCopy`, `DotProduct`, `VectorMA`, `VectorNormalize`, `AngleVectors`, `Square`, `CONTENTS_BODY`, etc.
- **`cm_patch.c`** — `CM_TraceThroughPatchCollide`, `CM_PositionTestInPatchCollide` (defined elsewhere)
- **`cm_test.c` / `cm_load.c`** — `CM_BoxLeafnums_r`, `CM_StoreLeafs`, `CM_ClipHandleToModel`, `CM_ModelBounds`, `CM_TempBoxModel` (defined elsewhere)
- **`c_traces`, `c_brush_traces`, `c_patch_traces`** — statistic counters defined in `cm_load.c`

# code/qcommon/cmd.c
## File Purpose
Implements Quake III's command buffer and command execution system. It manages a text-based FIFO buffer of pending console commands, tokenizes command strings into arguments, and dispatches commands to registered handlers, cvars, game modules, or the server.

## Core Responsibilities
- Maintain a fixed-size circular-style command text buffer (`cmd_text`)
- Provide `Cbuf_*` API to append, insert, and execute buffered command text
- Tokenize raw command strings into argc/argv-style argument arrays
- Register and unload named command functions via a linked list
- Dispatch commands to: registered handlers → cvars → cgame → game → UI → server forward
- Implement built-in commands: `cmdlist`, `exec`, `vstr`, `echo`, `wait`

## External Dependencies
- `../game/q_shared.h` — `byte`, `qboolean`, `MAX_STRING_TOKENS`, `BIG_INFO_STRING`, `MAX_QPATH`, `cbufExec_t`, `Q_strncpyz`, `COM_DefaultExtension`, `va`
- `qcommon.h` — `xcommand_t`, `Com_Printf`, `Com_Error`, `Cvar_Command`, `Cvar_VariableString`, `FS_ReadFile`, `FS_FreeFile`, `S_Malloc`, `Z_Free`, `CopyString`, `Com_Filter`
- **Defined elsewhere:** `CL_GameCommand`, `SV_GameCommand`, `UI_GameCommand`, `CL_ForwardCommandToServer`, `com_cl_running`, `com_sv_running`

# code/qcommon/common.c
## File Purpose
The central nervous system of Quake III Arena's engine, providing initialization, shutdown, per-frame orchestration, memory management (zone and hunk allocators), error handling, event loop, and shared utilities used by both client and server subsystems.

## Core Responsibilities
- Engine startup (`Com_Init`) and shutdown (`Com_Shutdown`) sequencing
- Per-frame loop (`Com_Frame`): event dispatch, server tick, client tick, timing
- Zone memory allocator (two pools: `mainzone`, `smallzone`) with tag-based freeing
- Hunk memory allocator (dual-ended stack: low/high with temp and permanent regions)
- Error handling (`Com_Error`) with `longjmp`-based recovery for non-fatal drops
- Event system: push/pop queue with optional journal file recording/replay
- Command-line parsing and startup variable injection
- Console tab-completion infrastructure

## External Dependencies
- `../game/q_shared.h`, `qcommon.h` — shared types, cvar, filesystem, net, VM interfaces
- `<setjmp.h>` — `setjmp`/`longjmp` for ERR_DROP recovery
- `<netinet/in.h>` (Linux/macOS) or `<winsock.h>` (Win32) — network byte order
- **Defined elsewhere (called here):** `CL_*`, `SV_*`, `FS_*`, `Cvar_*`, `Cmd_*`, `Cbuf_*`, `NET_*`, `VM_*`, `Sys_*`, `Netchan_Init`, `MSG_*`, `Key_WriteBindings`, `UI_usesUniqueCDKey`, `CIN_CloseAllVideos`

# code/qcommon/cvar.c
## File Purpose
Implements Quake III Arena's console variable (cvar) system, providing dynamic runtime configuration variables accessible from the console, config files, and C code. It manages cvar storage, lookup, value setting with protection flags, and VM-module bridging.

## Core Responsibilities
- Allocate and register cvars in a fixed-size pool with hash-table fast lookup
- Enforce protection flags: `CVAR_ROM`, `CVAR_INIT`, `CVAR_LATCH`, `CVAR_CHEAT`
- Track modification state per-cvar and globally via `cvar_modifiedFlags`
- Provide console commands: `toggle`, `set`, `sets`, `setu`, `seta`, `reset`, `cvarlist`, `cvar_restart`
- Serialize archived cvars to config file via `Cvar_WriteVariables`
- Bridge native cvars to VM (QVM) modules via `vmCvar_t` handle system
- Build info strings for userinfo/serverinfo/systeminfo network transmission

## External Dependencies
- **Includes:** `../game/q_shared.h` (types, `cvar_t`, `vmCvar_t`, flag constants), `qcommon.h` (Z_*, `CopyString`, `Com_*`, `Cmd_*`, `FS_Printf`, `Info_SetValueForKey*`)
- **Defined elsewhere:** `Z_Free`, `Z_Malloc`/`CopyString`, `Com_Error`, `Com_Printf`, `Com_DPrintf`, `Com_sprintf`, `Com_Filter`, `Cmd_Argc`, `Cmd_Argv`, `Cmd_AddCommand`, `FS_Printf`, `Info_SetValueForKey`, `Info_SetValueForKey_Big`, `Q_stricmp`, `Q_strncpyz`

# code/qcommon/files.c
## File Purpose
Implements Quake III Arena's handle-based virtual filesystem, which transparently merges content from multiple source directories and `.pk3` (zip) archives. It manages all file I/O for the engine, enforcing path security, pure-server validation, and demo/restricted-mode restrictions.

## Core Responsibilities
- Initialize and shut down the search path hierarchy (base/cd/home paths, mod directories)
- Load and index `.pk3` zip archives into hash-table-backed `pack_t` structures
- Resolve file reads by walking `fs_searchpaths` in priority order (pk3 before dir, newer pak before older)
- Enforce pure-server mode (only allow files from server-approved pak checksums)
- Track pak reference flags (`FS_GENERAL_REF`, `FS_CGAME_REF`, `FS_UI_REF`, `FS_QAGAME_REF`) for sv_pure negotiation
- Provide directory listing, mod enumeration, and file copy/rename operations
- Support journal-based replay of config file reads

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `fileHandle_t`, `cvar_t`, `fsMode_t`, `Q_str*` utilities
- `qcommon.h` — `Com_Error`, `Com_Printf`, `Cvar_Get`, `Cmd_*`, `Hunk_*`, `Z_Malloc/Free`, `S_ClearSoundBuffer`, `Com_BlockChecksum`
- `unzip.h` — `unzFile`, `unz_s`, `unzOpen/Close/Read/…` (zlib-based zip reading)
- **Defined elsewhere:** `Sys_ListFiles`, `Sys_FreeFileList`, `Sys_Mkdir`, `Sys_DefaultCDPath/InstallPath/HomePath`, `Sys_BeginStreamedFile/EndStreamedFile/StreamedRead/StreamSeek`, `Com_AppendCDKey`, `Com_ReadCDKey`, `Com_FilterPath`

# code/qcommon/huffman.c
## File Purpose
Implements an Adaptive Huffman compression/decompression codec used for network message encoding in Quake III Arena. Based on the algorithm from Sayood's *Data Compression* textbook, with node ranks implicitly encoded via doubly-linked list position rather than stored explicitly.

## Core Responsibilities
- Maintain and update an adaptive Huffman tree as symbols are transmitted/received
- Encode symbols to bit-stream output using prefix codes derived from tree position
- Decode symbols from bit-stream input by traversing the tree
- Compress/decompress full `msg_t` network message buffers
- Initialize persistent `huffman_t` state for use by the network channel layer

## External Dependencies
- `../game/q_shared.h` — `byte`, `qboolean`, `Com_Memset`, `Com_Memcpy`
- `qcommon.h` — `msg_t`, `node_t`, `huff_t`, `huffman_t`, `NYT`, `INTERNAL_NODE`, `HMAX` constants, all public `Huff_*` prototypes
- `oldsize` — declared `extern int`; defined elsewhere (likely `msg.c`); referenced but not used in this file's visible code paths

# code/qcommon/md4.c
## File Purpose
Implements the RSA Data Security MD4 message-digest algorithm, adapted for use in Quake III Arena's common layer. It provides two engine-facing checksum utilities built on top of the standard MD4 hash primitives.

## Core Responsibilities
- Define MD4 context type and initialize hash state
- Process arbitrary-length byte buffers through the MD4 compression function in 64-byte blocks
- Finalize a hash operation into a 16-byte digest with proper padding and bit-length encoding
- Expose `Com_BlockChecksum` and `Com_BlockChecksumKey` for engine-wide data integrity checks
- Encode/decode between little-endian byte arrays and 32-bit word arrays

## External Dependencies
- `<string.h>` — included at top (likely for `memset`/`memcpy` fallbacks).
- `Com_Memset`, `Com_Memcpy` — **defined elsewhere** (`qcommon/common.c`); used in place of `memset`/`memcpy` throughout. Under `__VECTORC` they alias to the standard functions directly.
- `#pragma warning(disable : 4711)` — MSVC-specific; suppresses inline expansion warnings on Windows builds.

# code/qcommon/msg.c
## File Purpose
Implements the network message serialization layer for Quake III Arena, providing bit-level read/write primitives over a `msg_t` buffer. It handles both raw out-of-band (OOB) byte-aligned I/O and Huffman-compressed bitstream I/O, and provides delta-compression for `usercmd_t`, `entityState_t`, and `playerState_t` structures.

## Core Responsibilities
- Initialize and manage `msg_t` buffers (normal and OOB modes)
- Write/read individual bits, bytes, shorts, longs, floats, strings, and angles
- Perform Huffman-compressed bit I/O via `msgHuff` global
- Delta-encode/decode `usercmd_t` (with optional XOR key obfuscation)
- Delta-encode/decode `entityState_t` using a static field descriptor table
- Delta-encode/decode `playerState_t` including fixed-size stat/ammo/powerup arrays
- Initialize the Huffman codec from a hardcoded byte-frequency table (`msg_hData`)

## External Dependencies
- **Includes:** `../game/q_shared.h`, `qcommon.h`
- **Defined elsewhere:** `Huff_Init`, `Huff_addRef`, `Huff_putBit`, `Huff_getBit`, `Huff_offsetTransmit`, `Huff_offsetReceive` (implemented in `huffman.c`); `cl_shownet` cvar (client module); `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy` (common); `LittleShort`, `LittleLong` (platform endian macros)

# code/qcommon/net_chan.c
## File Purpose
Implements the Quake III reliable sequenced network channel (`netchan_t`) layer, providing packet fragmentation/reassembly, out-of-order/duplicate suppression, and loopback routing. Also supplies address utility functions (`NET_CompareAdr`, `NET_AdrToString`, `NET_StringToAdr`) and out-of-band datagram helpers.

## Core Responsibilities
- Initialize and configure network channels (`Netchan_Init`, `Netchan_Setup`)
- Transmit messages, fragmenting payloads ≥ `FRAGMENT_SIZE` across multiple UDP packets
- Reassemble incoming fragments into a complete message buffer
- Discard duplicate and out-of-order packets; track dropped packet count
- Route loopback packets through in-process ring buffers instead of the OS socket
- Provide out-of-band text and binary datagram sending (`NET_OutOfBandPrint`, `NET_OutOfBandData`)
- Parse and format network addresses (`NET_StringToAdr`, `NET_AdrToString`)

## External Dependencies
- **Includes:** `../game/q_shared.h`, `qcommon.h`
- **Defined elsewhere:**
  - `MSG_*` — `code/qcommon/msg.c`
  - `Cvar_Get` — `code/qcommon/cvar.c`
  - `Sys_SendPacket`, `Sys_StringToAdr` — platform layer (`win32/`, `unix/`)
  - `Huff_Compress` — `code/qcommon/huffman.c`
  - `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy`, `Com_sprintf` — `code/qcommon/common.c`
  - `NET_Init`, `NET_Shutdown` — platform-specific net init (not in this file)

# code/qcommon/qcommon.h
## File Purpose
Central shared header for Quake III Arena's engine, declaring all subsystem interfaces shared between the client and server (but not game VM or renderer modules). It acts as the primary include for engine-level code, aggregating messaging, networking, VM, command, cvar, filesystem, memory, and platform abstraction APIs.

## Core Responsibilities
- Declares the `msg_t` bitstream serialization API (read/write primitives, delta compression)
- Declares the network layer: address types, packet I/O, `netchan_t` reliable sequenced channels
- Defines the protocol version, server/client opcode enums (`svc_ops_e`, `clc_ops_e`)
- Declares the Virtual Machine (`vm_t`) lifecycle and call interface
- Declares command buffer (`Cbuf_*`) and command execution (`Cmd_*`) APIs
- Declares the console variable (`Cvar_*`) system
- Declares the virtual filesystem (`FS_*`) with pk3/PAK abstraction
- Declares zone/hunk memory allocators (`Z_Malloc`, `Hunk_*`)
- Declares Adaptive Huffman compression structures and functions
- Declares platform abstraction (`Sys_*`) and client/server frame-loop entry points

## External Dependencies
- `code/qcommon/cm_public.h` → collision model public API (`CM_LoadMap`, `CM_BoxTrace`, etc.)
- `code/qcommon/qfiles.h` (via `cm_public.h`) → on-disk format structures
- `q_shared.h` (implicitly required) → `qboolean`, `vec3_t`, `cvar_t`, `fileHandle_t`, `vmCvar_t`, `usercmd_t`, `entityState_t`, `playerState_t`, `trace_t`, etc. — all defined elsewhere
- All `MSG_*`, `NET_*`, `Netchan_*`, `VM_*`, `Cmd_*`, `Cvar_*`, `FS_*`, `Com_*`, `Sys_*`, `Huff_*`, `Z_*`, `Hunk_*` function bodies are **defined elsewhere** in their respective `.c` files

# code/qcommon/qfiles.h
## File Purpose
Defines the on-disk binary file formats for all major Quake III Arena asset types: QVM bytecode, image formats (PCX, TGA), skeletal/rigid 3D models (MD3, MD4), and BSP map data. This header must remain identical between the engine and tool utilities to ensure consistent parsing.

## Core Responsibilities
- Define magic numbers, version constants, and hard limits for each file format
- Declare packed structs that directly map to serialized on-disk layouts
- Provide BSP lump index constants and the BSP header/lump descriptor types
- Define MD3 (rigid keyframe) and MD4 (skeletal/weighted) model structures
- Define surface geometry types and per-vertex draw data for the BSP renderer
- Establish world-space coordinate bounds and lightmap dimensions

## External Dependencies
- No `#include` directives in this file; consumers must include `q_shared.h` first to supply `vec2_t`, `vec3_t`, and `byte` typedefs used within these structs.
- All symbols are self-contained definitions; nothing here references external functions.

# code/qcommon/unzip.c
## File Purpose
A self-contained ZIP decompression library, adapted from zlib 1.1.3 / minizip 0.15 and embedded directly into the Quake III engine. It provides the `unzFile` API used by `files.c` to read game assets (`.pk3` files are ZIP archives). The entire zlib inflate pipeline — block processing, Huffman tree building, code decoding, Adler-32 checksumming — is inlined here as a single translation unit.

## Core Responsibilities
- Open, enumerate, and close ZIP archives via `unzOpen`/`unzClose`
- Navigate the ZIP central directory to locate specific files by name
- Open a specific file entry within a ZIP for streaming read (`unzOpenCurrentFile`)
- Decompress stored (method 0) or deflated (method 8) entries into caller-supplied buffers
- Provide the full zlib inflate state machine: `inflate`, `inflate_blocks`, `inflate_codes`, `inflate_fast`
- Build Huffman decode trees for dynamic and fixed deflate blocks (`huft_build`, `inflate_trees_*`)
- Replace zlib's `malloc`/`free` with Q3's `Z_Malloc`/`Z_Free` via `zcalloc`/`zcfree`

## External Dependencies
- `../client/client.h` → transitively pulls in `q_shared.h`, `qcommon.h` (for `Com_Memcpy`, `Com_Memset`, `Z_Malloc`, `Z_Free`, `LittleShort`, `LittleLong`)
- `unzip.h` — declares the public `unzFile` API types and error codes consumed by `files.c`
- **Defined elsewhere:** `Z_Malloc`, `Z_Free`, `Com_Memcpy`, `Com_Memset`, `LittleShort`, `LittleLong`, `Sys_Error` (debug only)
- No platform I/O beyond standard C `FILE*` (`fopen`, `fread`, `fseek`, `ftell`, `fclose`)

# code/qcommon/unzip.h
## File Purpose
Public header for a ZIP file reading library (unzip), adapted from the zlib/minizip project for use in Quake III Arena's virtual filesystem. It declares all types, internal structures, error codes, and the full API for opening, navigating, and decompressing entries within ZIP-format `.pk3` files.

## Core Responsibilities
- Define the opaque `unzFile` handle type (with optional strict-typing via `STRICTUNZIP`)
- Declare metadata structures for ZIP global info, per-file info, and date/time
- Expose the internal streaming state (`z_stream`, `file_in_zip_read_info_s`, `unz_s`) directly in the header
- Define error/status codes for all unzip operations
- Declare the full public API for ZIP navigation and decompression

## External Dependencies
- `<stdio.h>` — `FILE*` used directly in `unz_s` and `file_in_zip_read_info_s`
- `struct internal_state` — forward-declared; defined in zlib internals (`zconf.h`/`zlib.h`), not in this file
- `Z_ERRNO` — zlib error code macro, defined externally (zlib.h); aliased as `UNZ_ERRNO`
- Implementation defined in `code/qcommon/unzip.c`

# code/qcommon/vm.c
## File Purpose
Implements Quake III Arena's virtual machine management layer, supporting three execution modes: native DLL, compiled QVM bytecode, and interpreted QVM bytecode. It handles VM lifecycle (create, restart, free), cross-boundary call dispatch, symbol table loading, and developer profiling/info commands.

## Core Responsibilities
- Initialize and manage up to 3 simultaneous VM instances (cgame, game, ui)
- Load `.qvm` files from disk, validate headers, and allocate hunk memory for code/data
- Dispatch calls into VMs via `VM_Call`, routing to DLL entry point, compiled, or interpreted backend
- Bridge VM-to-engine system calls via `VM_DllSyscall`
- Translate VM integer pointers to host C pointers with optional mask-based bounds enforcement
- Load and walk `.map` symbol files for developer debugging and profiling
- Expose `vmprofile` and `vminfo` console commands

## External Dependencies
- `vm_local.h` → `q_shared.h`, `qcommon.h` — shared types, CVar/Cmd/FS/Hunk/Com APIs
- `Sys_LoadDll` / `Sys_UnloadDll` — platform DLL loader (defined in `sys_*` / `win_main.c` / `unix_main.c`)
- `VM_Compile` / `VM_CallCompiled` — defined in `vm_x86.c` (or `vm_ppc.c`, `vm_ppc_new.c`)
- `VM_PrepareInterpreter` / `VM_CallInterpreted` — defined in `vm_interpreted.c`
- `FS_ReadFile`, `FS_FreeFile`, `Hunk_Alloc`, `Z_Malloc`, `Z_Free`, `Com_Error`, `Com_Printf`, `Cvar_Get`, `Cmd_AddCommand` — all defined in other `qcommon/` modules

# code/qcommon/vm_interpreted.c
## File Purpose
Implements the software interpreter backend for the Quake III Q3VM virtual machine. It prepares bytecode for interpreted execution and runs the fetch-decode-execute loop over Q3VM instructions, supporting recursive entry and system call dispatch.

## Core Responsibilities
- Translate raw Q3VM bytecode into an int-aligned code image, resolving branch targets to absolute code offsets
- Execute Q3VM instructions via a central dispatch loop using a software operand stack
- Dispatch negative program counters as system calls to the engine
- Enforce VM sandboxing via `dataMask` on all memory accesses
- Support recursive VM entry (reentrant interpreter state per call)
- Provide debug utilities: stack trace, call indentation, opcode name table (when `DEBUG_VM` is defined)

## External Dependencies
- **Includes:** `vm_local.h` → `q_shared.h`, `qcommon.h`
- **Defined elsewhere:**
  - `vm_t`, `vmHeader_t`, `vmSymbol_t`, `opcode_t` — `vm_local.h`
  - `Hunk_Alloc`, `Com_Error`, `Com_Printf` — engine common layer
  - `VM_ValueToSymbol`, `VM_ValueToFunctionSymbol`, `VM_LogSyscalls`, `VM_Debug` — `vm.c`
  - `currentVM`, `vm_debugLevel` — `vm.c`
  - `loadWord` — macro; on PPC uses `lwbrx` byte-reverse load; on other platforms a plain int dereference

# code/qcommon/vm_local.h
## File Purpose
Internal header defining the data structures, opcodes, and function prototypes for Quake III's Virtual Machine (QVM) system. It is shared by the interpreter (`vm_interpreted.c`), JIT compiler (`vm_x86.c`, `vm_ppc.c`), and core VM manager (`vm.c`).

## Core Responsibilities
- Define the full QVM opcode set used by the bytecode interpreter and compiler
- Declare the `vm_s` (aka `vm_t`) structure holding all runtime state for a VM instance
- Declare the `vmSymbol_t` linked-list structure for debug symbol tracking
- Expose function prototypes for the two execution backends (compiled and interpreted)
- Expose symbol lookup utilities and syscall logging

## External Dependencies
- `../game/q_shared.h` — base types (`qboolean`, `byte`, `MAX_QPATH`, etc.)
- `qcommon.h` — `vm_t` forward declaration, `vmInterpret_t`, `vmHeader_t`, `VM_Create`/`VM_Free`/`VM_Call` public API
- `vmHeader_t` — defined in `qfiles.h` (via `qcommon.h`); not defined in this file
- `QDECL` — calling-convention macro from `q_shared.h`

# code/qcommon/vm_ppc.c
## File Purpose
Implements a dynamic JIT compiler that translates Quake III VM bytecode (Q3VM opcodes) into native PowerPC machine code at load time. It also provides the runtime entry point (`VM_CallCompiled`) and the system-call trampoline (`AsmCall`) needed to execute the generated code and dispatch engine syscalls.

## Core Responsibilities
- Translate Q3VM opcode stream into raw PPC 32-bit instructions written into a memory buffer
- Perform a multi-pass compile (3 passes: `-1`, `0`, `1`) so forward branch targets resolve correctly
- Emit peephole optimizations (e.g., collapsing CONST+LOAD pairs, eliding redundant stack pushes before binary ops)
- Allocate and finalize the native code buffer on the hunk after pass 0
- Provide `VM_CallCompiled` to set up the VM stack frame and jump into generated code
- Provide `AsmCall` (GCC inline asm or Metrowerks asm) to handle both intra-VM calls and engine syscall dispatch

## External Dependencies
- `vm_local.h` → `vm_t`, `vmHeader_t`, `opcode_t` enum, `currentVM`, `vm_debugLevel`
- `q_shared.h` / `qcommon.h` → `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `qboolean`, `byte`
- `AsmCall` — defined in this file, but its address is passed into generated code as a register constant
- `vm->systemCall` — defined elsewhere (engine syscall dispatcher, set at VM creation time)

# code/qcommon/vm_ppc_new.c
## File Purpose
Implements a PowerPC JIT compiler for Quake III's bytecode virtual machine. It translates Q3 VM bytecode (`vmHeader_t`) into native PPC machine code at load time, then provides an entry point (`VM_CallCompiled`) to execute that code natively.

## Core Responsibilities
- Translate Q3 VM opcodes to PPC machine instructions in a multi-pass compile loop
- Manage a virtual operand stack using physical PPC integer and float registers
- Emit properly encoded PPC instruction words (I-form, D-form, X-form, etc.)
- Patch load instructions retroactively to switch between integer (`LWZ/LWZX`) and float (`LFS/LFSX`) variants as operand types are resolved
- Set up and tear down the native stack frame on VM entry/exit (`OP_ENTER`/`OP_LEAVE`)
- Handle both VM-to-VM calls and VM-to-system-trap calls via `AsmCall`
- Provide `AsmCall` in inline GCC assembly (or CodeWarrior assembly) to dispatch both call types

## External Dependencies
- `vm_local.h` → `q_shared.h`, `qcommon.h`: `vm_t`, `vmHeader_t`, opcode enum, `Com_Error`, `Com_Printf`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`, `Com_Memcpy`, `currentVM`
- `AsmCall` — declared `extern void AsmCall(void)` and defined at file bottom; referenced by `VM_Compile` (address stored in r7/`R_ASMCALL` at runtime)
- `itofConvert` — file-static but referenced by address inside JIT-emitted code for `OP_CVIF`

# code/qcommon/vm_x86.c
## File Purpose
Implements a load-time x86 JIT compiler for Quake III's virtual machine bytecode. It translates Q3VM opcodes into native x86 machine code at load time, and provides the entry point for calling into the compiled code.

## Core Responsibilities
- Translate Q3VM bytecode (`vmHeader_t`) into native x86 machine code (`VM_Compile`)
- Perform peephole optimizations during two-pass compilation (e.g., folding CONST+LOAD, eliding redundant stack moves)
- Manage the `AsmCall` trampoline for VM-to-syscall and VM-to-VM dispatch
- Execute compiled VM code via `VM_CallCompiled`, setting up the program stack and opstack
- Track jump targets (`jused[]`) to prevent optimizations that cross branch destinations
- Handle cross-platform differences (Win32 `__declspec(naked)` vs GCC inline asm)

## External Dependencies
- `vm_local.h` → `q_shared.h`, `qcommon.h`: `vm_t`, `vmHeader_t`, `opcode_t`, `currentVM`, `Com_Error`, `Com_Printf`, `Com_Memcpy`, `Com_Memset`, `Z_Malloc`, `Z_Free`, `Hunk_Alloc`
- `sys/mman.h` (non-Win32, `mprotect`) — guarded out in released code (`#if 0`)
- `_ftol` (Win32 CRT) / `qftol0F7F` (Unix NASM, `unix/ftol.nasm`): float-to-int conversion
- `AsmCall` / `doAsmCall`: defined in this file but referenced via `asmCallPtr` indirection so the JIT-emitted `call` uses an indirect pointer fixup

# code/renderer/qgl.h
## File Purpose
This header defines the QGL abstraction layer — a cross-platform indirection layer over the OpenGL API. It provides `qgl*`-prefixed function pointers (or macros) that wrap every standard OpenGL 1.x function, enabling runtime dynamic loading on Windows/Linux and optional call logging/error-checking on macOS.

## Core Responsibilities
- Include the correct platform-specific GL headers (`<GL/gl.h>`, `<windows.h>+<gl/gl.h>`, or macOS frameworks) based on preprocessor guards.
- Declare `extern` function pointer variables for every core GL 1.x function on Windows and Linux (dynamic dispatch via `GetProcAddress`/`dlsym`).
- Declare `extern` function pointers for ARB multitexture and `EXT_compiled_vertex_array` extensions (`qglMultiTexCoord2fARB`, `qglActiveTextureARB`, `qglClientActiveTextureARB`, `qglLockArraysEXT`, `qglUnlockArraysEXT`).
- On non-Windows/non-macOS platforms, redirect all `qgl*` names directly to `gl*` via `#define` macros (`qgl_linked.h`).
- On macOS, include `macosx_qgl.h` which provides static-inline wrappers with optional per-call logging (`QGL_LOG_GL_CALLS`) and error checking (`QGL_CHECK_GL_ERRORS`).
- On Windows, additionally declare `qwgl*` function pointers for the WGL context/pixel-format API and swap-interval extension.
- On Linux/FreeBSD, additionally declare `qglX*` function pointers for the GLX API and optionally `qfxMesa*` for 3Dfx Glide.

## External Dependencies

- `<GL/gl.h>`, `<GL/glx.h>` — system OpenGL headers (Linux/FreeBSD)
- `<windows.h>`, `<gl/gl.h>` — Windows OpenGL headers
- `<OpenGL/gl.h>`, `<OpenGL/glu.h>`, `<OpenGL/glext.h>` — macOS OpenGL framework (via `macosx_glimp.h`)
- `<GL/fxmesa.h>` — 3Dfx Mesa extension (Linux, conditional on `__FX__`)
- `code/renderer/qgl_linked.h` — `#define qgl* gl*` redirects for statically-linked platforms
- `code/macosx/macosx_qgl.h` — autogenerated macOS inline wrappers with debug instrumentation
- `code/macosx/macosx_glimp.h` — macOS GL include aggregator
- `qwglGetProcAddress`, `glXGetProcAddress`, `dlsym` etc. — **defined elsewhere** (platform `win_qgl.c` / `linux_qgl.c`); used at renderer init to populate these pointers

# code/renderer/qgl_linked.h
## File Purpose
This header provides a compile-time macro mapping layer that aliases all `qgl*` OpenGL wrapper function names directly to their native `gl*` counterparts. It is used on platforms where OpenGL is statically linked (e.g., macOS), eliminating the need for runtime function pointer indirection.

## Core Responsibilities
- Maps every `qgl*` call used in the renderer to the corresponding standard `gl*` OpenGL 1.x function via `#define`
- Provides a zero-overhead, compile-time alternative to the dynamic dispatch path used in `qgl.h` / `linux_qgl.c` / `win_qgl.c`
- Covers the full OpenGL 1.1 core API surface including geometry, texturing, state management, display lists, feedback, evaluators, and pixel operations
- Enables the rest of the renderer to use `qgl*` names uniformly regardless of platform linking strategy

## External Dependencies
- Implicitly depends on a system OpenGL header (e.g., `<GL/gl.h>` or `<OpenGL/gl.h>`) being included before or alongside this file to supply the `gl*` symbol declarations.
- All `gl*` symbols referenced are **defined elsewhere** — provided by the platform OpenGL library (e.g., `libGL.so`, `OpenGL.framework`, `opengl32.dll`).

# code/renderer/tr_animation.c
## File Purpose
Implements skeletal animation (MD4 model format) for the Quake III renderer. It handles both the front-end surface submission and the back-end per-frame vertex skinning via weighted bone transforms.

## Core Responsibilities
- Register MD4 animated surfaces into the draw surface list (front-end)
- Interpolate bone matrices between two animation frames (lerp)
- Deform mesh vertices using weighted, multi-bone skeletal skinning
- Write skinned positions, normals, and texture coordinates into the tessellator (`tess`)
- Copy triangle index data into the tessellator index buffer

## External Dependencies
- **Includes:** `tr_local.h` (brings in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `md4Header_t`, `md4Surface_t`, `md4LOD_t`, `md4Frame_t`, `md4Bone_t`, `md4Vertex_t`, `md4Weight_t` — defined in `qfiles.h`
  - `tess` (`shaderCommands_t`) — defined/owned by `tr_shade.c` / `tr_backend.c`
  - `backEnd` (`backEndState_t`) — defined in `tr_backend.c`
  - `tr` (`trGlobals_t`) — defined in `tr_init.c`
  - `R_AddDrawSurf`, `R_GetShaderByHandle`, `RB_CheckOverflow` — defined in other renderer modules
  - `VectorClear`, `DotProduct` — macros from `q_shared.h`

# code/renderer/tr_backend.c
## File Purpose
This is the OpenGL render back end for Quake III Arena. It executes a command queue of render operations issued by the front end, manages OpenGL state transitions, and drives the actual draw calls for 3D surfaces and 2D UI elements.

## Core Responsibilities
- Maintain and cache OpenGL state (texture bindings, blend modes, depth, cull, alpha test) to minimize redundant API calls
- Execute the render command queue (`RC_SET_COLOR`, `RC_STRETCH_PIC`, `RC_DRAW_SURFS`, `RC_DRAW_BUFFER`, `RC_SWAP_BUFFERS`, `RC_SCREENSHOT`)
- Iterate sorted draw surfaces per-frame, batching by shader/fog/entity/dlight
- Set up per-entity model-view matrices and dynamic lighting transforms
- Support 2D orthographic rendering (UI, cinematics, stretch-pic)
- Handle SMP: optionally run the back end on a dedicated render thread

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `rb_surfaceTable[]` — surface dispatch table (defined in `tr_surface.c`)
  - `tess` (`shaderCommands_t`) — tesselator globals
  - `tr` (`trGlobals_t`), `glConfig`, `glState` — renderer globals
  - `RB_BeginSurface`, `RB_EndSurface`, `RB_CheckOverflow` — tesselator (`tr_shade.c`)
  - `R_DecomposeSort`, `R_RotateForEntity`, `R_TransformDlights` — front-end math helpers
  - `RB_ShadowFinish`, `RB_RenderFlares`, `RB_TakeScreenshotCmd` — other back-end modules
  - `GLimp_*` — platform-specific GL window/thread layer
  - `ri` (`refimport_t`) — engine import table (memory, print, time)

# code/renderer/tr_bsp.c
## File Purpose
Loads a Quake III BSP map file from disk and converts all its lumps into renderer-ready in-memory structures. It is the single entry point for world map loading (`RE_LoadWorldMap`) and handles all surface types, lightmaps, visibility data, fog volumes, BSP nodes/leaves, and the volumetric light grid.

## Core Responsibilities
- Parse and byte-swap all BSP lumps into the `s_worldData` (`world_t`) structure
- Upload lightmap textures to GPU with overbright color shifting
- Convert on-disk surfaces (`dsurface_t`) to typed render surfaces: planar faces, patch meshes, triangle soups, and flares
- Pre-tessellate Bezier patch meshes and stitch/fix LOD cracks between adjacent patches
- Build the BSP node/leaf tree with parent links for PVS traversal
- Load fog volumes, planes, shader references, visibility clusters, and the ambient light grid
- Allocate all world geometry into the engine hunk

## External Dependencies
- **Includes:** `tr_local.h` → `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`
- **Defined elsewhere:**
  - `tr`, `ri`, `glConfig` — renderer globals
  - `R_SubdividePatchToGrid`, `R_GridInsertColumn`, `R_GridInsertRow`, `R_FreeSurfaceGridMesh` — `tr_curve.c`
  - `R_FindShader`, `R_RemapShader` — `tr_shader.c`
  - `R_CreateImage`, `R_SyncRenderThread` — `tr_image.c` / `tr_init.c`
  - `R_AllocModel` — `tr_model.c`
  - `COM_ParseExt`, `COM_Parse`, `LittleLong`, `LittleFloat` — `qcommon`
  - `r_vertexLight`, `r_lightmap`, `r_mapOverBrightBits`, `r_fullbright`, `r_singleShader` — cvars registered in `tr_init.c`

# code/renderer/tr_cmds.c
## File Purpose
This file implements the renderer's command buffer system, acting as the bridge between the front-end (scene submission) and back-end (GPU execution) render threads. It manages double-buffered render command lists and supports optional SMP (symmetric multiprocessing) via a dedicated render thread.

## Core Responsibilities
- Initialize and shut down the SMP render thread
- Provide a command buffer allocation mechanism (`R_GetCommandBuffer`)
- Enqueue typed render commands (draw surfaces, set color, stretch pic, draw buffer, swap buffers)
- Issue buffered commands to the back end (single-threaded or SMP wake)
- Synchronize front and back end threads before mutating shared GL state
- Display per-frame performance counters at various verbosity levels

## External Dependencies
- `tr_local.h` — all renderer types, globals (`tr`, `backEnd`, `glConfig`, `glState`), cvars, SMP platform functions
- `GLimp_SpawnRenderThread`, `GLimp_FrontEndSleep`, `GLimp_WakeRenderer` — platform-specific SMP primitives (defined in `win_glimp.c` / `linux_glimp.c`)
- `RB_ExecuteRenderCommands`, `RB_RenderThread` — defined in `tr_backend.c`
- `R_ToggleSmpFrame`, `R_SumOfUsedImages`, `R_SetColorMappings`, `GL_TextureMode` — defined elsewhere in the renderer

# code/renderer/tr_curve.c
## File Purpose
Converts raw Bézier patch control-point grids (read from map data) into subdivided `srfGridMesh_t` render surfaces. It handles adaptive LOD subdivision, normal generation, patch stitching via column/row insertion, and mesh lifecycle (alloc/free).

## Core Responsibilities
- Adaptively subdivide a patch mesh in both axes based on `r_subdivisions` error tolerance
- Compute per-vertex normals accounting for mesh wrapping and degenerate edges
- Cull collinear rows/columns from the final grid
- Allocate and populate `srfGridMesh_t` with LOD error tables and bounding data
- Free `srfGridMesh_t` allocations (supports `PATCH_STITCHING` heap path)
- Insert a new column or row into an existing grid (patch stitching)
- Optionally transpose grid for longer triangle strips

## External Dependencies
- **`tr_local.h`** — pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`
- **`ri`** (`refimport_t`) — `ri.Malloc`, `ri.Free`, `ri.Hunk_Alloc` for memory management
- **`r_subdivisions`** (`cvar_t *`) — controls adaptive subdivision error threshold
- **`MAX_GRID_SIZE` (65), `MAX_PATCH_SIZE` (32)** — defined in `tr_local.h`
- **`srfGridMesh_t`, `drawVert_t`, `SF_GRID`** — defined in `tr_local.h` / `qfiles.h`
- **Vector math macros** (`VectorSubtract`, `CrossProduct`, etc.) — defined in `q_shared.h`
- **`Com_Memcpy`, `Com_Memset`** — defined elsewhere in qcommon

# code/renderer/tr_flares.c
## File Purpose
Implements the light flare rendering subsystem for Quake III Arena's renderer. Flares simulate an ocular effect where bright light sources produce visible glare rings; they use depth buffer readback to determine visibility and interpolate intensity across frames for smooth fading.

## Core Responsibilities
- Maintain a pool of `flare_t` state objects across multiple frames and scenes
- Project 3D flare positions to screen-space coordinates during surface tessellation
- Read back the depth buffer per-flare to test occlusion after opaque geometry is drawn
- Fade flare intensity smoothly in/out using time-based interpolation
- Render each visible flare as a screen-aligned quad in orthographic projection
- Register dynamic light sources (dlights) as flares via `RB_AddDlightFlares`

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `backEnd`, `tr`, `glState` — renderer globals
  - `tess` (`shaderCommands_t`) — tessellator buffer
  - `r_flares`, `r_flareSize`, `r_flareFade` — cvars
  - `R_TransformModelToClip`, `R_TransformClipToWindow` — `tr_main.c`
  - `RB_BeginSurface`, `RB_EndSurface` — `tr_shade.c`
  - `qglReadPixels`, `qglOrtho`, etc. — QGL wrappers

# code/renderer/tr_font.c
## File Purpose
Implements the font registration and rendering system for Quake III Arena's renderer. It supports both runtime TrueType rasterization via FreeType 2 (compile-time opt-in via `BUILD_FREETYPE`) and the standard path of loading pre-rendered glyph bitmaps and atlas textures from disk.

## Core Responsibilities
- Load pre-rendered font `.dat` files and associated TGA atlas images from `fonts/`
- Cache up to `MAX_FONTS` registered fonts to avoid redundant loads
- Register glyph shader handles via `RE_RegisterShaderNoMip` for each loaded font
- (When `BUILD_FREETYPE`) Rasterize TrueType glyphs using FreeType, pack them into 256×256 GL texture pages, and optionally write `.dat`/`.tga` output files
- Provide endian-safe binary deserialization helpers (`readInt`, `readFloat`) for `.dat` files
- Initialize and shut down the FreeType library (`R_InitFreeType`, `R_DoneFreeType`)

## External Dependencies
- `tr_local.h` — renderer internals: `image_t`, `ri`, `R_SyncRenderThread`, `R_CreateImage`, `RE_RegisterShaderNoMip`, `RE_RegisterShaderFromImage`, `r_saveFontData`
- `qcommon/qcommon.h` — `Z_Malloc`, `Z_Free`, `Com_Memset`, `Com_Memcpy`, `Com_sprintf`, `Q_stricmp`, `Q_strncpyz`
- `fontInfo_t`, `glyphInfo_t`, `GLYPHS_PER_FONT`, `GLYPH_START`, `GLYPH_END` — defined elsewhere (likely `game/q_shared.h` or `qcommon/qfiles.h`)
- FreeType 2 headers (`ft2/freetype.h`, etc.) — only when `BUILD_FREETYPE` is defined; not shipped in release builds

# code/renderer/tr_image.c
## File Purpose
This file implements the renderer's complete image management system for Quake III Arena, handling loading, processing, uploading, and caching of all game textures. It supports BMP, PCX, TGA, and JPEG formats, manages OpenGL texture objects, and owns the skin registration system.

## Core Responsibilities
- Load raw image data from disk in multiple formats (BMP, PCX, TGA, JPEG)
- Resample, mipmap, and gamma/intensity-correct images before GPU upload
- Upload processed pixel data to OpenGL via `qglTexImage2D`
- Cache loaded images in a hash table to avoid redundant loads
- Create and manage procedural built-in textures (dlight, fog, default, white)
- Manage skin (`.skin` file) registration and lookup
- Build gamma/intensity lookup tables used during texture upload

## External Dependencies
- `tr_local.h` — `image_t`, `tr`, `glConfig`, `glState`, `ri` (refimport), all renderer cvars
- `../jpeg-6/jpeglib.h` — libjpeg-6 compression/decompression API (included with `JPEG_INTERNALS` defined)
- OpenGL via QGL wrappers: `qglTexImage2D`, `qglTexParameterf`, `qglDeleteTextures`, `qglBindTexture`, `qglActiveTextureARB`
- `GL_Bind`, `GL_SelectTexture`, `GL_CheckErrors`, `GLimp_SetGamma` — defined in other renderer files
- `R_FindShader`, `R_SyncRenderThread` — defined in `tr_shader.c` / render thread code
- `ri.FS_ReadFile`, `ri.FS_FreeFile`, `ri.Malloc`, `ri.Free`, `ri.Hunk_Alloc`, `ri.Hunk_AllocateTempMemory`, `ri.Hunk_FreeTempMemory`, `ri.Error`, `ri.Printf` — engine import table

# code/renderer/tr_init.c
## File Purpose
This is the renderer initialization and shutdown module for Quake III Arena's OpenGL renderer. It registers all renderer cvars, initializes the OpenGL subsystem and renderer subsystems (images, shaders, models), and exposes the renderer's public API via `GetRefAPI`.

## Core Responsibilities
- Declare and define all renderer `cvar_t*` globals used across the renderer module
- Register all renderer cvars with the engine cvar system in `R_Register`
- Initialize OpenGL via `InitOpenGL` (calls platform `GLimp_Init`, sets default GL state)
- Initialize renderer subsystems: images, shaders, skins, models, FreeType, function tables
- Allocate SMP-aware back-end data buffers (`backEndData[0/1]`)
- Handle screenshot capture (TGA and JPEG) via a render command queue
- Provide `GetRefAPI` — the DLL entry point that returns the `refexport_t` vtable to the engine

## External Dependencies
- `tr_local.h` — all renderer-internal types, globals, and function declarations
- `GLimp_Init` / `GLimp_Shutdown` — platform-specific GL window creation (defined in `win_glimp.c` / `linux_glimp.c` / `macosx_glimp.m`)
- `SaveJPG` — JPEG encoder (defined in `tr_image.c`)
- `ri` (`refimport_t`) — engine callbacks for cvars, commands, file I/O, memory, printing (defined elsewhere, imported via `GetRefAPI`)
- `R_InitImages`, `R_InitShaders`, `R_InitSkins`, `R_ModelInit`, `R_InitFreeType` — defined in their respective `tr_*.c` files
- `R_InitCommandBuffers`, `R_ToggleSmpFrame` — defined in `tr_cmds.c`
- `R_InitFogTable`, `R_NoiseInit` — defined in `tr_noise.c` / fog subsystem

# code/renderer/tr_light.c
## File Purpose
Handles dynamic and static lighting calculations for the Quake III Arena renderer. It computes per-entity lighting by sampling the world light grid (trilinear interpolation) and accumulating dynamic light contributions, then stores results used by the shader backend.

## Core Responsibilities
- Transform dynamic light (dlight) origins into local entity space
- Determine which dlights intersect a bmodel's bounding box and mark affected surfaces
- Sample the world light grid via trilinear interpolation to compute ambient and directed light for entities
- Accumulate dlight contributions into per-entity lighting vectors
- Expose a public API for querying lighting at an arbitrary world point

## External Dependencies
- `tr_local.h` — all renderer types, `tr`, `backEnd`, `ri`, math macros
- `VectorSubtract`, `DotProduct`, `VectorCopy`, `VectorClear`, `VectorMA`, `VectorScale`, `VectorNormalize`, `VectorNormalize2`, `VectorLength` — defined in shared math library
- `Com_Memset` — defined in `qcommon`
- `ri.Printf` — renderer import table, defined elsewhere
- `myftol` — fast float-to-int, platform-specific (inline or asm)
- `r_ambientScale`, `r_directedScale`, `r_debugLight` — cvars registered in `tr_init.c`
- `tr.sinTable` — precomputed in `tr_init.c`
- `FUNCTABLE_SIZE`, `FUNCTABLE_MASK` — constants from `tr_local.h`

# code/renderer/tr_local.h
## File Purpose
This is the primary internal header for the Quake III Arena renderer module. It defines all renderer-private data structures, global state, constants, and function prototypes used across the renderer's front-end and back-end subsystems. No external code outside the renderer should include this file.

## Core Responsibilities
- Define all renderer-internal types: shaders, surfaces, models, textures, lights, fog, world BSP structures
- Declare the two major global singletons: `tr` (front-end globals) and `backEnd` (back-end state)
- Define the `shaderCommands_t` tesselator (`tess`) used by the back-end to batch geometry
- Declare the render command queue types and SMP double-buffering structures
- Expose all internal function prototypes grouped by subsystem (shaders, world, lights, curves, skies, etc.)
- Declare all renderer cvars as `extern cvar_t *`
- Define GL state abstraction types (`glstate_t`, `GLS_*` bit flags)

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `cplane_t`, `qboolean`, `cvar_t`, `refEntity_t`, etc.
- `../qcommon/qfiles.h` — `md3Header_t`, `md4Header_t`, `drawVert_t`, `dshader_t`, BSP lump types, `SHADER_MAX_VERTEXES`
- `../qcommon/qcommon.h` — `refimport_t`, memory allocators, filesystem, cvar/cmd APIs
- `tr_public.h` — `refexport_t`, `refimport_t`, `glconfig_t`, `stereoFrame_t` (from `cgame/tr_types.h`)
- `qgl.h` — `qgl*` function pointer wrappers for OpenGL
- `GLimp_*` functions — platform-specific GL window/thread management (defined in `win32/` or `unix/`)
- `SHADER_MAX_VERTEXES` / `SHADER_MAX_INDEXES` — defined in `qfiles.h`, constrain `shaderCommands_t` arrays

# code/renderer/tr_main.c
## File Purpose
This is the main control-flow file for the renderer front end, responsible for per-frame view setup, frustum culling, draw-surface submission and sorting, portal/mirror view recursion, and dispatching entity surfaces to the back-end command queue.

## Core Responsibilities
- Build and set up view-space orientation matrices (`R_RotateForViewer`, `R_RotateForEntity`)
- Compute and set the perspective projection matrix and far-clip distance
- Derive frustum planes for view-space culling
- Perform AABB and sphere frustum culling (`R_CullLocalBox`, `R_CullPointAndRadius`)
- Handle portal and mirror surface detection, orientation computation, and recursive view rendering
- Collect and sort all draw surfaces for a frame (`R_AddDrawSurf`, `R_SortDrawSurfs`)
- Dispatch sorted surfaces to the render back end via `R_AddDrawSurfCmd`

## External Dependencies
- **`tr_local.h`** — all renderer types, cvar externs, and subsystem prototypes
- **`q_shared.h` / `qcommon.h`** — math primitives (`VectorMA`, `DotProduct`, `PlaneFromPoints`, `PerpendicularVector`, `CrossProduct`, `RotatePointAroundVector`, `SetPlaneSignbits`)
- **`RB_BeginSurface`, `rb_surfaceTable`** — back-end surface tessellation (defined in `tr_backend.c` / `tr_surface.c`)
- **`R_AddWorldSurfaces`, `R_AddPolygonSurfaces`** — world and polygon surface adders (defined in `tr_world.c` / `tr_scene.c`)
- **`R_AddMD3Surfaces`, `R_AddAnimSurfaces`, `R_AddBrushModelSurfaces`** — model-type-specific surface adders (defined elsewhere in renderer)
- **`R_AddDrawSurfCmd`** — enqueues the sorted surface list to the back-end command buffer (defined in `tr_cmds.c`)
- **`R_SyncRenderThread`** — SMP render-thread synchronization (defined in `tr_init.c` or platform layer)
- **`ri.CM_DrawDebugSurface`** — collision-map debug callback (defined in collision module)
- **`tess`** (`shaderCommands_t`) — global tessellator state (defined in `tr_shade.c`)

# code/renderer/tr_marks.c
## File Purpose
Implements polygon projection ("marks") onto world geometry for decal-like effects such as bullet holes and scorch marks. It traverses the BSP tree to collect candidate surfaces, clips projected polygons against those surfaces, and returns fragments for use by the cgame.

## Core Responsibilities
- Traverse the BSP tree to collect surfaces within an AABB (`R_BoxSurfaces_r`)
- Clip a polygon against a half-space plane (`R_ChopPolyBehindPlane`)
- Clip surface triangles against the projection volume's bounding planes (`R_AddMarkFragments`)
- Project a mark polygon onto planar (`SF_FACE`) and curved grid (`SF_GRID`) world surfaces (`R_MarkFragments`)
- Filter surfaces by shader flags (`SURF_NOIMPACT`, `SURF_NOMARKS`, `CONTENTS_FOG`) and face angle relative to projection direction

## External Dependencies
- **Includes:** `tr_local.h` (all renderer types, `trGlobals_t tr`, math macros)
- **Defined elsewhere:** `tr` (`trGlobals_t`), `BoxOnPlaneSide`, `DotProduct`, `CrossProduct`, `VectorNormalize2`, `VectorNormalizeFast`, `VectorMA`, `VectorAdd`, `VectorSubtract`, `VectorCopy`, `VectorInverse`, `ClearBounds`, `AddPointToBounds`, `Com_Memcpy`, `markFragment_t`, `srfSurfaceFace_t`, `srfGridMesh_t`, `drawVert_t`, `SURF_NOIMPACT`, `SURF_NOMARKS`, `CONTENTS_FOG`, `VERTEXSIZE`

# code/renderer/tr_mesh.c
## File Purpose
Handles front-end rendering of MD3 triangle mesh models, including culling, LOD selection, fog membership, and submission of draw surfaces to the renderer's sort queue.

## Core Responsibilities
- Cull MD3 models against the view frustum using bounding spheres and boxes
- Compute the appropriate LOD level based on projected screen-space radius
- Determine which fog volume (if any) the model occupies
- Resolve the correct shader per surface (custom shader, skin, or embedded MD3 shader)
- Submit shadow draw surfaces (stencil and projection) for opaque surfaces
- Submit main draw surfaces to `R_AddDrawSurf` for deferred sorting and rendering
- Skip "personal model" (RF_THIRD_PERSON) surfaces unless rendering through a portal

## External Dependencies
- **Includes:** `tr_local.h` (transitively includes `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `R_CullLocalPointAndRadius`, `R_CullLocalBox` — `tr_main.c`
  - `R_AddDrawSurf` — `tr_main.c`
  - `R_SetupEntityLighting` — `tr_light.c`
  - `R_GetShaderByHandle`, `R_GetSkinByHandle` — `tr_shader.c` / `tr_image.c`
  - `RadiusFromBounds` — `q_shared.c` / math library
  - `myftol` — platform-specific (x86 asm or cast macro)
  - `tr`, `r_lodscale`, `r_lodbias`, `r_shadows` — `tr_init.c` / `tr_main.c`

# code/renderer/tr_model.c
## File Purpose
Handles loading, caching, and querying of 3D models for the Quake III renderer. Supports two model formats: MD3 (static mesh with per-frame vertex animation) and MD4 (skeletal/bone-weighted mesh). Also provides tag interpolation and model bounds queries used during entity rendering.

## Core Responsibilities
- Allocate and register models into the global `tr.models[]` registry via handle
- Load and byte-swap MD3 files, including multi-LOD variants (up to `MD3_MAX_LODS`)
- Load and byte-swap MD4 files, including their LOD surfaces and bone-weighted vertices
- Register shaders referenced by model surfaces during load
- Interpolate MD3 attachment tags between animation frames (`R_LerpTag`)
- Provide model AABB bounds for culling (`R_ModelBounds`)
- Initialize the model subsystem and expose a debug listing command

## External Dependencies
- **`tr_local.h`** — pulls in all renderer types, `tr` globals, `ri` refimport, `qfiles.h` MD3/MD4 structs
- `ri.Hunk_Alloc`, `ri.FS_ReadFile`, `ri.FS_FreeFile`, `ri.Printf`, `ri.Error` — engine syscalls via `refimport_t ri`
- `R_FindShader` — defined in `tr_shader.c`
- `R_Init`, `R_SyncRenderThread`, `R_ClearFlares`, `RE_ClearScene`, `RE_StretchPic` — defined elsewhere in the renderer
- `Com_Memcpy`, `Q_strncpyz`, `Q_strlwr`, `VectorNormalize`, `AxisClear`, `VectorClear` — shared utilities from `q_shared.c`/`q_math.c`

# code/renderer/tr_noise.c
## File Purpose
Implements a 4-dimensional value noise generator for the Quake III Arena renderer. It provides seeded random noise lookup and trilinear+temporal interpolation used by shader effects such as waveform deformations and turbulence.

## Core Responsibilities
- Initialize a fixed-size noise table and permutation array with a deterministic seed
- Provide a permutation-indexed lookup into the noise table via `INDEX`/`VAL` macros
- Perform 4D trilinear interpolation over the noise lattice (x, y, z, t)
- Expose `R_NoiseInit` and `R_NoiseGet4f` as the renderer's public noise API

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Standard library:** `floor` (via math.h transitively), `srand`, `rand`
- **Defined elsewhere:** `R_NoiseInit` and `R_NoiseGet4f` are declared in `tr_local.h` and called by other renderer modules.

# code/renderer/tr_public.h
## File Purpose
Defines the public ABI boundary between the Quake III renderer module and the engine/client. It declares two function-pointer structs (`refexport_t` and `refimport_t`) and the single DLL entry point `GetRefAPI`, enabling the renderer to be loaded as a dynamically swappable module.

## Core Responsibilities
- Define `REF_API_VERSION` for compatibility checking at load time
- Declare all renderer-exported functions via `refexport_t` (scene building, resource registration, frame control, etc.)
- Declare all engine services imported by the renderer via `refimport_t` (memory, filesystem, cvars, commands, etc.)
- Expose `GetRefAPI` as the sole linker-visible symbol for module initialization
- Include `tr_types.h` to bring shared render types into scope

## External Dependencies

- **`../cgame/tr_types.h`** — shared render types: `refEntity_t`, `refdef_t`, `polyVert_t`, `glconfig_t`, `stereoFrame_t`, `markFragment_t`, `fontInfo_t`, `orientation_t`, `qhandle_t`, `refEntityType_t`, etc.
- **Defined elsewhere:** `vec3_t`, `qboolean`, `byte`, `cvar_t`, `ha_pref`, `e_status`, `QDECL`, `BIG_INFO_STRING`, `MAX_STRING_CHARS`, `MAX_MAP_AREA_BYTES` — all from `q_shared.h` or platform headers pulled in transitively.
- **`__USEA3D`** — conditional A3D audio-geometry hook; platform-specific, not cross-platform.

# code/renderer/tr_scene.c
## File Purpose
Implements the renderer's scene submission API, acting as the front-end interface between the game/cgame modules and the renderer pipeline. It accumulates entities, dynamic lights, and polygons into double-buffered back-end data arrays, then triggers a view render pass via `RE_RenderScene`.

## Core Responsibilities
- Toggle SMP (symmetric multi-processing) double-buffer frames and reset scene counters
- Accept and buffer `refEntity_t` submissions into `backEndData` entity arrays
- Accept and buffer dynamic light submissions (normal and additive) into `backEndData` dlight arrays
- Accept and buffer client-submitted polygons (`srfPoly_t`) into `backEndData` poly/polyVert arrays, including fog volume assignment
- Flush all buffered polygon surfaces into the current view's draw surface list
- Populate `tr.refdef` from the `refdef_t` descriptor and invoke `R_RenderView` to execute a 3D render pass
- Support multiple scenes per frame (3D game view, HUD models, menus) via `firstScene*` offset bookkeeping

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:** `tr` (`trGlobals_t`), `backEndData` (`backEndData_t*[SMP_FRAMES]`), `glConfig` (`glconfig_t`), `ri` (`refimport_t`), `r_smp`, `r_norefresh`, `r_dynamiclight`, `r_vertexLight` (cvars), `max_polys`, `max_polyverts`, `R_RenderView`, `R_AddDrawSurf`, `R_GetShaderByHandle`, `AddPointToBounds`, `GLimp_LogComment`

# code/renderer/tr_shade.c
## File Purpose
This is the renderer back end's surface shading module. It applies shader programs (multi-stage, multi-pass) to tessellated surface geometry stored in the global `tess` struct, dispatching to OpenGL draw calls with appropriate texture, color, and blend state.

## Core Responsibilities
- Initialize and finalize per-surface tessellation batches (`RB_BeginSurface`, `RB_EndSurface`)
- Compute per-vertex colors (`ComputeColors`) and texture coordinates (`ComputeTexCoords`) for each shader stage
- Dispatch geometry to OpenGL via triangle strips or indexed triangles (`R_DrawElements`, `R_DrawStripElements`)
- Handle multi-pass rendering: generic stages, vertex-lit, lightmapped multitexture
- Apply dynamic light projections as additive/modulate passes (`ProjectDlightTexture`)
- Apply fog blending pass (`RB_FogPass`)
- Support debug visualization of triangle wireframes and vertex normals

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `backEnd`, `tr`, `glConfig`, `glState` — renderer globals
  - `r_primitives`, `r_logFile`, `r_lightmap`, `r_showtris`, `r_shownormals`, `r_vertexLight`, `r_uiFullScreen`, `r_offsetFactor`, `r_offsetUnits`, `r_debugSort` — cvars
  - `RB_DeformTessGeometry`, all `RB_Calc*` functions — defined in `tr_shade_calc.c`
  - `RB_ShadowTessEnd` — defined in `tr_shadows.c`
  - `GL_Bind`, `GL_State`, `GL_Cull`, `GL_SelectTexture`, `GL_TexEnv` — `tr_init.c` / `tr_main.c`
  - `GLimp_LogComment` — platform-specific implementation
  - `ri.CIN_RunCinematic`, `ri.CIN_UploadCinematic`, `ri.Error` — engine import table

# code/renderer/tr_shade_calc.c
## File Purpose
Implements the shader calculation support functions for the Quake III renderer back end, providing vertex deformation, color generation, alpha generation, and texture coordinate generation. All functions operate on the global tessellator buffer (`tess`) and are called during shader stage evaluation before geometry is submitted to OpenGL.

## Core Responsibilities
- Evaluate waveform functions (sin, triangle, square, sawtooth, noise) against precomputed lookup tables
- Deform tessellated vertex positions and normals (wave, bulge, move, autosprite, text)
- Generate per-vertex colors from entity properties, waveforms, and diffuse lighting
- Generate per-vertex alpha values from entity properties, waveforms, and specular calculation
- Generate and transform texture coordinates (environment mapping, fog, turbulence, scroll, scale, rotate, stretch)
- Apply fog density modulation to per-vertex color and alpha channels

## External Dependencies
- **`tr_local.h`** — all renderer types, `tess` (shaderCommands_t), `backEnd`, `tr`, `ri`
- **Defined elsewhere:** `R_NoiseGet4f`, `RB_AddQuadStamp`, `RB_AddQuadStampExt`, `RB_CalcFogTexCoords` (self-referential within file), `RB_CalcTransformTexCoords`, `RB_ProjectionShadowDeform`, `Q_rsqrt`, `VectorNormalizeFast`, `VectorNormalize`, `myftol` (x86 inline asm or macro fallback)
- **`WAVEVALUE` macro** — inline table lookup combining phase, time, and frequency into a table index

# code/renderer/tr_shader.c
## File Purpose
Parses and manages all shader definitions for the Quake III Arena renderer. It handles loading `.shader` text files, parsing their syntax into `shader_t`/`shaderStage_t` structures, optimizing multi-pass shaders (multitexture collapsing, vertex lighting), and maintaining a hash-table registry of all loaded shaders.

## Core Responsibilities
- Load and concatenate all `.shader` script files from the `scripts/` directory into a single in-memory text buffer
- Parse shader text blocks into the global `shader`/`stages` workspace, then promote to permanent hunk-allocated instances
- Resolve shader lookups by name and lightmap index, creating implicit default shaders for unmapped images
- Optimize shaders: collapse two-pass modulate/add combos into single multitexture passes; apply vertex-lighting collapse when hardware demands it
- Maintain two hash tables: one for registered `shader_t*` instances, one for fast text-block lookup by name
- Provide public registration entry points (`RE_RegisterShader`, `RE_RegisterShaderLightMap`, `RE_RegisterShaderNoMip`)
- Remap shaders at runtime via `R_RemapShader`
- Fix in-flight render command lists when new shaders shift sorted indices

## External Dependencies
- `tr_local.h` — all renderer types (`shader_t`, `shaderStage_t`, `trGlobals_t tr`, `glConfig`, `ri`, cvars)
- `COM_ParseExt`, `SkipRestOfLine`, `SkipBracedSection`, `COM_Compress`, `COM_StripExtension`, `COM_DefaultExtension` — defined in `qcommon`
- `R_FindImageFile`, `R_InitSkyTexCoords` — defined in `tr_image.c` / `tr_sky.c`
- `RB_StageIteratorGeneric`, `RB_StageIteratorSky`, `RB_StageIteratorVertexLitTexture`, `RB_StageIteratorLightmappedMultitexture` — defined in `tr_shade.c`
- `R_SyncRenderThread` — defined in `tr_backend.c`
- `ri.Hunk_Alloc`, `ri.FS_ListFiles`, `ri.FS_ReadFile`, `ri.CIN_PlayCinematic` — engine import table (`refimport_t ri`)
- `backEndData`, `tr`, `glConfig` — renderer globals defined in `tr_init.c` / `tr_main.c`

# code/renderer/tr_shadows.c
## File Purpose
Implements real-time shadow rendering for Quake III Arena's renderer backend. Provides two shadow techniques: stencil-buffer shadow volumes (for per-entity silhouette shadows) and flat projection shadow deformation (for planar blob shadows cast onto surfaces).

## Core Responsibilities
- Build per-vertex edge definition lists from tessellated geometry
- Determine which triangles face the light source
- Identify silhouette edges (edges shared only by front-facing triangles)
- Render shadow volume geometry into the stencil buffer (increment/decrement passes)
- Apply a full-screen darkening quad to pixels marked by the stencil buffer
- Deform vertex positions to project geometry flat onto a shadow plane for projection shadows

## External Dependencies
- **Includes:** `tr_local.h` (pulls in all renderer types, `tess`, `backEnd`, `tr`, `glConfig`, `r_shadows`)
- **Defined elsewhere:** `tess` (`shaderCommands_t`), `backEnd` (`backEndState_t`), `tr.whiteImage`, `glConfig.stencilBits`, `r_shadows` cvar, all `qgl*` OpenGL wrappers, math macros (`VectorCopy`, `VectorMA`, `DotProduct`, `CrossProduct`, `VectorSubtract`), `Com_Memset`, `GL_Bind`, `GL_State`

# code/renderer/tr_sky.c
## File Purpose
Implements sky and cloud rendering for Quake III Arena's renderer backend. It handles sky polygon clipping to a cube box, generation of subdivided sky box geometry, cloud layer vertex generation with spherical projection, and sun quad rendering.

## Core Responsibilities
- Clip world-space sky polygons onto the 6 faces of a sky cube box to determine which sky face regions need drawing
- Generate subdivided mesh vertices and texture coordinates for the sky box outer shell
- Compute cloud layer texture coordinates using a spherical intersection formula (called once at shader parse time)
- Populate `tess` (the tessellator) with cloud geometry vertices and indices per-frame
- Draw the sky box outer faces directly via immediate-mode OpenGL (`qglBegin`/`qglEnd`)
- Render the sun as a billboard quad aligned to `tr.sunDirection`
- Act as the sky shader stage iterator (`RB_StageIteratorSky`), orchestrating the full sky draw sequence

## External Dependencies
- **`tr_local.h`** — all renderer types, `tess`, `backEnd`, `tr`, cvars (`r_fastsky`, `r_drawSun`, `r_showsky`)
- **Defined elsewhere:** `RB_StageIteratorGeneric`, `RB_BeginSurface`, `RB_EndSurface`, `GL_Bind`, `GL_State`, `PerpendicularVector`, `CrossProduct`, `Q_acos`, `myftol`, `ri.Error`, all `qgl*` OpenGL wrappers

# code/renderer/tr_surface.c
## File Purpose
Implements the renderer back-end surface tessellation dispatch layer for Quake III Arena. It converts every recognized surface type (BSP faces, grid meshes, triangle soups, MD3 meshes, sprites, beams, rails, lightning) into vertices and indices written into the global `tess` (shaderCommands_t) buffer for subsequent shader execution.

## Core Responsibilities
- Guard the tess buffer against overflow and flush/restart it via `RB_CheckOverflow`
- Emit billboard quads (sprites, flares) into the tess buffer
- Tessellate static BSP geometry: planar faces (`srfSurfaceFace_t`), grid/patch meshes (`srfGridMesh_t`), triangle soups (`srfTriangles_t`)
- Lerp and decode MD3 compressed vertex/normal data into the tess buffer (`LerpMeshVertexes`, `RB_SurfaceMesh`)
- Generate procedural geometry for special entity types: beams, rail core/rings, lightning bolts
- Dispatch to the correct tessellation function through the `rb_surfaceTable` function pointer array

## External Dependencies
- `tr_local.h` — all renderer types, `tess`, `backEnd`, `tr`, cvar externs
- `RB_BeginSurface`, `RB_EndSurface` — defined in `tr_cmds.c`/`tr_backend.c`
- `RB_SurfaceAnim` — defined in `tr_animation.c` (MD4)
- `GL_Bind`, `GL_State`, `qglBegin/End/Vertex/Color` — OpenGL wrappers; `RB_SurfaceBeam` and `RB_SurfaceAxis` bypass the tess buffer and issue immediate-mode GL directly
- `r_railWidth`, `r_railCoreWidth`, `r_railSegmentLength`, `r_lodCurveError` — cvars read at draw time
- `PerpendicularVector`, `RotatePointAroundVector`, `MakeNormalVectors`, `VectorNormalizeFast` — defined in `q_math.c`

# code/renderer/tr_world.c
## File Purpose
Implements the renderer front-end world traversal for Quake III Arena. It walks the BSP tree to determine which world surfaces are potentially visible this frame, culls them against the view frustum and PVS, and submits them to the draw surface list. It also handles brush model surface submission and dynamic light (dlight) intersection testing.

## Core Responsibilities
- Mark visible BSP leaves via PVS/areamask (`R_MarkLeaves`)
- Recursively traverse the BSP tree with frustum culling (`R_RecursiveWorldNode`)
- Cull individual surfaces (face, grid, triangle) before submission
- Distribute dlight bits down the BSP tree and per-surface
- Submit visible surfaces to the renderer sort list via `R_AddDrawSurf`
- Handle brush model (inline model) surface submission separately
- Provide `R_inPVS` utility for visibility queries between two points

## External Dependencies
- **`tr_local.h`** — all renderer types, globals (`tr`, `backEnd`), cvars, and function prototypes
- **Defined elsewhere:**
  - `R_CullLocalBox`, `R_CullPointAndRadius`, `R_CullLocalPointAndRadius` — `tr_main.c`
  - `R_AddDrawSurf` — `tr_main.c`
  - `R_DlightBmodel` — `tr_light.c`
  - `R_GetModelByHandle` — `tr_model.c`
  - `BoxOnPlaneSide`, `DotProduct`, `ClearBounds` — math/shared utilities
  - `CM_ClusterPVS` — collision map module (`qcommon`)
  - `ri.Error`, `ri.Printf` — platform import table

# code/server/server.h
## File Purpose
Central header for the Quake III Arena dedicated server subsystem. Defines all major server-side data structures, global state variables, and declares the full public API surface across all server `.c` modules (`sv_main`, `sv_init`, `sv_client`, `sv_snapshot`, `sv_game`, `sv_bot`, `sv_world`, `sv_net_chan`).

## Core Responsibilities
- Define the per-frame server state (`server_t`) and persistent cross-map server state (`serverStatic_t`)
- Define per-client state (`client_t`), per-snapshot state (`clientSnapshot_t`), and connection handshake state (`challenge_t`)
- Define the server-side entity wrapper (`svEntity_t`) used for spatial partitioning and PVS/cluster tracking
- Declare all server cvars as extern pointers
- Declare all inter-module function prototypes for the server subsystem
- Expose the spatial world-query API (link/unlink, area queries, traces, point contents)

## External Dependencies
- `../game/q_shared.h` — shared types: `vec3_t`, `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t`, `cvar_t`, `netadr_t`, etc.
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `vm_t`, `PACKET_BACKUP`, `MAX_MSGLEN`, filesystem, cvar, cmd APIs
- `../game/g_public.h` — `sharedEntity_t`, `entityShared_t`, `gameImport_t`/`gameExport_t` trap enums, `SVF_*` flags
- `../game/bg_public.h` — `pmove_t`, game constants, configstring index definitions
- **Defined elsewhere:** `worldSector_s` (sv_world.c), `cmodel_s` (collision model system), all `SV_*` function bodies across `sv_*.c` files, `vm_t` (vm.c)

# code/server/sv_bot.c
## File Purpose
Serves as the server-side bridge between the Quake III game server and the BotLib AI library. It implements the `botlib_import_t` interface (callbacks the bot library calls into the engine) and exposes server-facing bot management functions for client slot allocation, per-frame ticking, and debug visualization.

## Core Responsibilities
- Allocate and free pseudo-client slots for bot entities
- Implement all `botlib_import_t` callbacks (trace, PVS, memory, file I/O, print, debug geometry)
- Initialize and populate the `botlib_import_t` vtable, then call `GetBotLibAPI` to obtain `botlib_export_t`
- Register all bot-related cvars at startup
- Drive the bot AI frame tick via `VM_Call(gvm, BOTAI_START_FRAME, time)`
- Provide bots access to reliable command queues and snapshot entity lists
- Manage a debug polygon pool for AAS visualization

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- `botlib.h` — defines `botlib_import_t`, `botlib_export_t`, `bsp_trace_t`, `bot_input_t`
- **Defined elsewhere:** `SV_Trace`, `SV_ClipToEntity`, `SV_PointContents`, `SV_inPVS`, `SV_ExecuteClientCommand`, `SV_GentityNum`, `CM_EntityString`, `CM_InlineModel`, `CM_ModelBounds`, `RadiusFromBounds`, `Z_TagMalloc`, `Z_Free`, `Z_Malloc`, `Z_AvailableMemory`, `Hunk_Alloc`, `Hunk_CheckMark`, `VM_Call`, `GetBotLibAPI`, `Sys_CheckCD`, `Cvar_Get`, `Cvar_VariableIntegerValue`, `Cvar_VariableValue`, `FS_FOpenFileByMode`, `FS_Read2`, `FS_Write`, `FS_FCloseFile`, `FS_Seek`, `gvm` (game VM handle), `svs`/`sv` server state globals.

# code/server/sv_ccmds.c
## File Purpose
Implements operator/admin console commands for the Quake III Arena server. These commands are restricted to stdin or remote operator datagrams and cover server management: map loading, player kicking/banning, status reporting, and server lifecycle control.

## Core Responsibilities
- Register all server operator commands via `SV_AddOperatorCommands`
- Resolve clients by name or slot number for targeted operations
- Load and restart maps (including single-player, devmap, and warmup-delayed restarts)
- Kick and ban players by name or client number
- Print server status, serverinfo, systeminfo, and per-user info to the console
- Broadcast console chat messages to all connected clients
- Force the next heartbeat to fire immediately

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `svs`, `sv`, `gvm`, `sv_maxclients`, `sv_gametype`, `sv_mapname`, `com_sv_running`, `com_frameTime`, `com_dedicated`; `SV_SpawnServer`, `SV_DropClient`, `SV_ClientEnterWorld`, `SV_RestartGameProgs`, `SV_AddServerCommand`, `SV_SendServerCommand`, `SV_SetConfigstring`, `SV_GameClientNum`, `SV_SectorList_f`; `VM_Call`, `VM_ExplicitArgPtr`; `NET_StringToAdr`, `NET_OutOfBandPrint`, `NET_AdrToString`; `Cmd_*`, `Cvar_*`, `Info_Print`, `FS_ReadFile`

# code/server/sv_client.c
## File Purpose
Handles all server-side client lifecycle management for Quake III Arena, from initial connection negotiation and authorization through in-game command processing, file downloads, and disconnection. It is the primary interface between raw network messages from clients and the game VM.

## Core Responsibilities
- Challenge/response handshake to prevent spoofed connections (`SV_GetChallenge`, `SV_AuthorizeIpPacket`)
- Direct connection processing: protocol validation, challenge verification, slot allocation (`SV_DirectConnect`)
- Client state transitions: `CS_FREE` → `CS_CONNECTED` → `CS_PRIMED` → `CS_ACTIVE` → `CS_ZOMBIE`
- Gamestate serialization and transmission to newly connected/map-restarted clients (`SV_SendClientGameState`)
- In-game packet parsing: client commands, user movement, flood protection (`SV_ExecuteClientMessage`)
- Pure server pak checksum validation (`SV_VerifyPaks_f`)
- Sliding-window file download streaming (`SV_WriteDownloadToClient`)
- Client disconnection and cleanup (`SV_DropClient`)

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `svs` (`serverStatic_t`), `sv` (`server_t`), `gvm` (`vm_t*`), all `sv_*` cvars, `VM_Call`, `Netchan_Setup`, `NET_OutOfBandPrint`, `FS_SV_FOpenFileRead`, `FS_Read`, `FS_idPak`, `FS_LoadedPakPureChecksums`, `MSG_*` family, `SV_Heartbeat_f`, `SV_SendClientSnapshot`, `SV_BotFreeClient`, `SV_GentityNum`

# code/server/sv_game.c
## File Purpose
This file implements the server-side interface between the Quake III engine and the game VM (virtual machine). It exposes engine services to the game DLL/bytecode through a system call dispatch table, and manages game VM lifecycle (init, restart, shutdown).

## Core Responsibilities
- Dispatch all game VM system calls via `SV_GameSystemCalls` (the single entry point for VM→engine calls)
- Translate between game-VM entity indices and server-side entity/client pointers
- Manage game VM lifecycle: load (`SV_InitGameProgs`), restart (`SV_RestartGameProgs`), shutdown (`SV_ShutdownGameProgs`)
- Forward bot library calls from the game VM to `botlib_export`
- Provide PVS (Potentially Visible Set) visibility tests for game logic
- Expose server state (serverinfo, userinfo, configstrings, usercmds) to the game VM

## External Dependencies
- `server.h` — `svs`, `sv`, `gvm`, all server types and function declarations
- `../game/botlib.h` — `botlib_export_t`, all `BOTLIB_*` syscall constants
- **Defined elsewhere:** `VM_Create`, `VM_Call`, `VM_Free`, `VM_Restart`, `VM_ArgPtr`; all `CM_*` collision functions; `SV_LinkEntity`, `SV_UnlinkEntity`, `SV_Trace`, `SV_AreaEntities`; `SV_BotAllocateClient`, `SV_BotLibSetup`, `SV_BotGetSnapshotEntity`; `BotImport_DebugPolygonCreate/Delete`; `FS_*`, `Cvar_*`, `Cmd_*`, `Cbuf_*`, `Com_*`, `Sys_*`, `MatrixMultiply`, `AngleVectors`, `PerpendicularVector`

# code/server/sv_init.c
## File Purpose
Handles server initialization, map spawning, and shutdown for Quake III Arena. It manages configstring/userinfo get/set operations, client array allocation, and the full lifecycle of loading a new map while transitioning connected clients into the new game state.

## Core Responsibilities
- Register and initialize all server-side cvars at engine startup (`SV_Init`)
- Manage configstring storage and reliable broadcast to connected clients (`SV_SetConfigstring`)
- Allocate/reallocate the `svs.clients` array on startup or `sv_maxclients` change
- Execute the full map spawn sequence: clear state, load BSP, init game VM, settle frames, create delta baselines
- Transition existing connected clients (human and bot) into the new level
- Send final disconnect messages to all clients on server shutdown

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `Z_Free`, `Z_Malloc`, `CopyString`, `Hunk_Alloc/Clear/SetMark`, `Hunk_AllocateTempMemory`, `VM_Call`, `VM_ExplicitArgPtr`, `FS_Restart`, `FS_ClearPakReferences`, `FS_LoadedPakChecksums/Names`, `FS_ReferencedPakChecksums/Names`, `CM_LoadMap`, `CM_ClearMap`, `CL_MapLoading`, `CL_ShutdownAll`, `CL_Disconnect`, `Cvar_Get/Set/VariableValue/InfoString/InfoString_Big`, `Com_Printf/Error/Milliseconds/Memset`, `SV_InitGameProgs`, `SV_ShutdownGameProgs`, `SV_ClearWorld`, `SV_SendServerCommand`, `SV_SendClientSnapshot`, `SV_DropClient`, `SV_BotFrame`, `SV_BotInitCvars`, `SV_BotInitBotLib`, `SV_Heartbeat_f`, `SV_MasterShutdown`, `SV_AddOperatorCommands`, `SV_RemoveOperatorCommands`

# code/server/sv_main.c
## File Purpose
Core server frame driver and network dispatch hub for Quake III Arena. It owns the two primary server-side globals (`svs`, `sv`, `gvm`), drives the per-frame game simulation loop, and routes all incoming UDP packets — both connectionless and in-sequence — to appropriate handlers.

## Core Responsibilities
- Define and expose all server-side cvars
- Manage reliable server-command queuing per client (`SV_AddServerCommand`, `SV_SendServerCommand`)
- Send/receive heartbeats to/from master servers
- Respond to connectionless queries: `getstatus`, `getinfo`, `getchallenge`, `connect`, `rcon`, `ipAuthorize`
- Dispatch sequenced in-game packets to the correct `client_t` via `SV_PacketEvent`
- Run the main server frame: ping calculation, timeout detection, game VM tick, snapshot dispatch, heartbeat

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `SV_DropClient`, `SV_GetChallenge`, `SV_DirectConnect`, `SV_AuthorizeIpPacket`, `SV_ExecuteClientMessage`, `SV_Netchan_Process`, `SV_BotFrame`, `SV_SendClientMessages`, `SV_SetConfigstring`, `SV_GameClientNum`, `VM_Call`, `NET_OutOfBandPrint`, `NET_StringToAdr`, `NET_Sleep`, `Huff_Decompress`, `Com_BeginRedirect`, `Com_EndRedirect`, `Cbuf_AddText`, `Cvar_InfoString`, `Cvar_InfoString_Big`, `cvar_modifiedFlags`, `com_dedicated`, `com_sv_running`, `cl_paused`, `sv_paused`, `com_speeds`, `time_game`

# code/server/sv_net_chan.c
## File Purpose
Provides server-side network channel wrapper functions that layer XOR-based obfuscation encoding/decoding on top of the base `Netchan` fragmentation and sequencing layer. It also manages a per-client outgoing message queue to prevent UDP packet bursts when large fragmented messages collide during transmission.

## Core Responsibilities
- XOR-encode outgoing server messages using client challenge, sequence number, and acknowledged command strings as a rolling key
- XOR-decode incoming client messages using matching key material
- Queue outgoing messages when the netchan already has unsent fragments, ensuring correct ordering
- Drain the outgoing queue by encoding and transmitting the next queued message once fragmentation completes
- Wrap `Netchan_Process` with a decode step for all received client packets

## External Dependencies
- `../game/q_shared.h` — base types (`byte`, `qboolean`, `msg_t` primitives)
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `Netchan_Transmit`, `Netchan_TransmitNextFragment`, `Netchan_Process`, `MSG_ReadLong`, `MSG_WriteByte`, `MSG_Copy`, `Z_Malloc`, `Z_Free`, `Com_DPrintf`, `Com_Error`; constants `SV_ENCODE_START`, `SV_DECODE_START`, `MAX_RELIABLE_COMMANDS`, `svc_EOF`
- `server.h` — `client_t`, `netchan_buffer_t`, `MAX_MSGLEN`
- `Netchan_*` functions — defined in `qcommon/net_chan.c` (not this file)

# code/server/sv_rankings.c
## File Purpose
Implements the server-side interface to Id Software's Global Rankings (GRank) system, managing player authentication, match tracking, and stat reporting via an external rankings API. It bridges Quake III Arena's server loop with the asynchronous GRank library using callback-based operations.

## Core Responsibilities
- Initialize and shut down the GRank rankings session per game match
- Authenticate players via server-side login/create or client-side token validation
- Track per-player GRank contexts, match handles, and player IDs
- Submit integer and string stat reports for players/server during gameplay
- Handle asynchronous GRank callbacks for new game, login, join game, send reports, and cleanup
- Encode/decode player IDs and tokens using a custom 6-bit ASCII encoding scheme
- Manage context reference counting to safely free resources when all contexts close

## External Dependencies
- `server.h` — server types, cvars (`sv_maxclients`, `sv_enableRankings`, `sv_rankingsActive`), `SV_SetConfigstring`, `Z_Malloc`, `Z_Free`, `Cvar_Set`, `Cvar_VariableValue`, `Com_DPrintf`
- `../rankings/1.0/gr/grapi.h` — GRank API: `GRankInit`, `GRankNewGameAsync`, `GRankUserLoginAsync`, `GRankUserCreateAsync`, `GRankJoinGameAsync`, `GRankPlayerValidate`, `GRankSendReportsAsync`, `GRankCleanupAsync`, `GRankStartMatch`, `GRankReportInt`, `GRankReportStr`, `GRankPoll`; types `GR_CONTEXT`, `GR_STATUS`, `GR_PLAYER_TOKEN`, `GR_NEWGAME`, `GR_LOGIN`, `GR_JOINGAME`, `GR_MATCH`, `GR_INIT` — **defined in external rankings library, not in this file**
- `../rankings/1.0/gr/grlog.h` — `GRankLogLevel`, `GRLOG_OFF`, `GRLOG_TRACE` — **defined in external rankings library**
- `LittleLong64` — byte-order conversion for 64-bit values — **defined elsewhere in qcommon**

# code/server/sv_snapshot.c
## File Purpose
Builds per-client game snapshots each server frame and transmits them over the network using delta compression. It determines entity visibility via PVS/area checks, encodes state deltas, and throttles transmission via rate control.

## Core Responsibilities
- Build `clientSnapshot_t` frames by culling visible entities per PVS and area connectivity
- Delta-encode entity states (`entityState_t` list) between frames for bandwidth efficiency
- Delta-encode `playerState_t` between frames
- Write the full snapshot packet (header, areabits, playerstate, entities) to a `msg_t`
- Retransmit unacknowledged reliable server commands to clients
- Throttle snapshot delivery using per-client rate and `snapshotMsec` limits
- Drive the per-frame send loop across all connected clients

## External Dependencies
- **Includes:** `server.h` → `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:**
  - `MSG_WriteDeltaEntity`, `MSG_WriteDeltaPlayerstate`, `MSG_WriteByte/Long/Bits/Data/String`, `MSG_Init`, `MSG_Clear` — `qcommon/msg.c`
  - `CM_PointLeafnum`, `CM_LeafArea`, `CM_LeafCluster`, `CM_ClusterPVS`, `CM_AreasConnected`, `CM_WriteAreaBits` — `qcommon/cm_*.c`
  - `SV_Netchan_Transmit`, `SV_Netchan_TransmitNextFragment` — `sv_net_chan.c`
  - `SV_GentityNum`, `SV_GameClientNum`, `SV_SvEntityForGentity` — `sv_game.c`
  - `SV_WriteDownloadToClient` — `sv_client.c`
  - `svs`, `sv`, `sv_padPackets`, `sv_maxRate`, `sv_lanForceRate`, `sv_maxclients` — globals/cvars

# code/server/sv_world.c
## File Purpose
Implements server-side spatial partitioning and world query operations for Quake III Arena. It maintains an axis-aligned BSP sector tree for fast entity lookups and provides collision tracing, area queries, and point-contents testing against both world geometry and game entities.

## Core Responsibilities
- Build and manage a uniform spatial subdivision tree (`worldSector_t`) for entity bucketing
- Link/unlink game entities into the sector tree when they move or change bounds
- Compute and cache PVS cluster memberships and area numbers per entity on link
- Query all entities whose AABBs overlap a given region (`SV_AreaEntities`)
- Perform swept-box traces through the world and all solid entities (`SV_Trace`)
- Clip a movement against a single specific entity (`SV_ClipToEntity`)
- Return combined content flags at a world point across all overlapping entities (`SV_PointContents`)

## External Dependencies
- **`server.h`** → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `CM_InlineModel`, `CM_ModelBounds`, `CM_BoxLeafnums`, `CM_LeafArea`, `CM_LeafCluster`, `CM_BoxTrace`, `CM_TransformedBoxTrace`, `CM_TransformedPointContents`, `CM_PointContents`, `CM_TempBoxModel`; `SV_SvEntityForGentity`, `SV_GEntityForSvEntity`, `SV_GentityNum`; `RadiusFromBounds`; globals `sv`, `svs`

# code/ui/keycodes.h
## File Purpose
Defines the canonical enumeration of all input key codes used by the Quake III Arena UI and input systems. It provides a hardware-agnostic numeric identity for every bindable input, including keyboard keys, mouse buttons, scroll wheel, joystick buttons, and auxiliary inputs.

## Core Responsibilities
- Define `keyNum_t`, the master enum of all recognized input identifiers
- Anchor ASCII-compatible keys at their literal ASCII values (Tab=9, Enter=13, Escape=27, Space=32)
- Enumerate extended keys (function keys, numpad, arrows, modifiers) starting at 128
- Enumerate mouse, scroll wheel, joystick (32 buttons), and auxiliary (16) inputs
- Define `K_CHAR_FLAG` bitmask to distinguish character events from key events in the menu system
- Assert via comment that `K_LAST_KEY` must remain below 256

## External Dependencies
- No includes. Self-contained.
- `keyNum_t` values are consumed by:
  - `KeyEvent()` — defined elsewhere in the client/input layer
  - Key-binding tables in `cl_keys.c`
  - Menu input handlers in `ui_main.c` / `ui_atoms.c`

# code/ui/ui_atoms.c
## File Purpose
Provides foundational UI utility functions for the Quake III Arena menu system, including drawing primitives, coordinate scaling, console command dispatch, and post-game score tracking/persistence.

## Core Responsibilities
- Bridges `q_shared.c` error/print functions to UI trap calls (when not hard-linked)
- Scales 640×480 virtual coordinates to actual screen resolution
- Dispatches UI console commands (`postgame`, `ui_cache`, `remapShader`, etc.)
- Persists and loads per-map post-game best scores to/from `.game` files
- Provides primitive 2D drawing helpers (filled rects, outlines, named/handle pics)
- Manages the `m_entersound` flag for menu interaction audio

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`, `ui_shared.h`
- `trap_*` functions — all defined in `ui_syscalls.c`; bridge to engine VM syscalls
- `uiInfo` (global `uiInfo_t`) — defined in `ui_main.c`
- `Display_CacheAll` — defined in `ui_shared.c`
- `UI_ShowPostGame`, `UI_Report`, `UI_Load` — defined in `ui_main.c`
- `Com_sprintf`, `Q_strncpyz`, `Q_stricmp`, `Info_ValueForKey` — defined in `q_shared.c`

# code/ui/ui_gameinfo.c
## File Purpose
Loads and parses arena map and bot definition files (`.arena`, `.bot`, `arenas.txt`, `bots.txt`) into global UI-accessible arrays. Provides lookup functions for bot info strings and populates `uiInfo.mapList` with parsed arena metadata used by the UI menu system.

## Core Responsibilities
- Parse key-value info blocks from arena/bot text files via `UI_ParseInfos`
- Load all arena definitions from `scripts/arenas.txt` and `*.arena` files into `ui_arenaInfos[]`
- Load all bot definitions from `scripts/bots.txt` and `*.bot` files into `ui_botInfos[]`
- Populate `uiInfo.mapList[]` with map name, load name, image path, and game-type bitfields
- Provide bot lookup by index or name
- Respect `g_arenasFile` / `g_botsFile` cvars to override default paths

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_shared.h`
- **Defined elsewhere:** `COM_Parse`, `COM_ParseExt`, `COM_Compress`, `Info_SetValueForKey`, `Info_ValueForKey`, `Q_strncpyz`, `Q_stricmp`, `UI_Alloc`, `UI_OutOfMemory`, `String_Alloc`, all `trap_*` syscall wrappers, `uiInfo` global, `MAX_BOTS`, `MAX_ARENAS`, `MAX_ARENAS_TEXT`, `MAX_BOTS_TEXT`, `MAX_MAPS`, game-type enum constants (`GT_FFA`, etc.)

# code/ui/ui_local.h
## File Purpose
This is the primary internal header for the Quake III Arena UI VM module. It aggregates all type definitions, constants, extern declarations, and trap (syscall) function prototypes needed by the UI subsystem's implementation files.

## Core Responsibilities
- Declares all `vmCvar_t` globals used across UI screens (game rules, server browser, scores, etc.)
- Defines the legacy `menuframework_s` / `menucommon_s` widget type system and associated flags
- Declares the `uiStatic_t` singleton holding frame-level UI state and asset handles
- Declares the large `uiInfo_t` aggregate holding all new-UI runtime state (server lists, maps, tiers, players, mods, demos)
- Provides the complete `trap_*` syscall interface the UI VM uses to call into the engine
- Forward-declares every UI screen module's public cache/init/display functions

## External Dependencies
- `../game/q_shared.h` — base types (`vec3_t`, `qboolean`, `vmCvar_t`, etc.)
- `../cgame/tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, etc.)
- `ui_public.h` — exported UI entry point enum (`uiExport_t`), `uiMenuCommand_t`, `uiClientState_t`
- `keycodes.h` — `keyNum_t` enum
- `../game/bg_public.h` — `weapon_t`, `animation_t`, `animNumber_t`, game type enums
- `ui_shared.h` — new-UI `displayContextDef_t` and shared menu-def types (defined elsewhere)
- Engine syscall dispatch — all `trap_*` targets are defined in the engine, not this module

# code/ui/ui_main.c
## File Purpose
The primary entry point and master controller for Quake III Arena's Team Arena UI module. It implements the `vmMain` dispatch function (the QVM entry point), manages all menu data, handles owner-draw rendering, input routing, server browser logic, and asset lifecycle for the entire UI system.

## Core Responsibilities
- Dispatch all UI VM commands via `vmMain` (init, shutdown, key/mouse events, refresh, active menu)
- Initialize and wire the `displayContextDef_t` function table with UI callbacks during `_UI_Init`
- Render per-frame UI: paint menus, draw cursor, update server/player lists via `_UI_Refresh`
- Implement all owner-draw items (handicap, player model, clan logo, map preview, team slots, etc.)
- Manage server browser: refresh, display list construction, binary insertion sorting, find-player searches
- Parse game data files: `gameinfo.txt`, `teaminfo.txt`, map lists, game types, character/alias tables
- Register and update all UI cvars through a static `cvarTable[]` descriptor array
- Execute menu scripts (`UI_RunMenuScript`) for game start, server join, bot add, settings changes

## External Dependencies
- `ui_local.h` → `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `ui_shared.h`, `keycodes.h`
- **Defined elsewhere:** `Menu_Count`, `Menu_PaintAll`, `Menu_GetFocused`, `Menu_HandleKey`, `Menu_New`, `Menu_Reset`, `Menu_SetFeederSelection`, `Menus_*`, `Display_*`, `Init_Display`, `String_*`, `Controls_*`, `UI_DrawPlayer`, `UI_PlayerInfo_*`, `UI_RegisterClientModelname`, `UI_LoadArenas`, `UI_LoadBestScores`, `UI_ClearScores`, `UI_LoadBots`, `UI_GetBotNameByNumber`, `UI_GetNumBots`, `trap_*` syscalls (all defined in `ui_syscalls.c`)

# code/ui/ui_players.c
## File Purpose
Handles 3D player model rendering and animation state management for the Quake III Arena UI. Provides the `UI_DrawPlayer` function used to display animated player characters in menus (character selection, player settings, etc.), along with model/skin/animation loading utilities.

## Core Responsibilities
- Load and register player model parts (legs, torso, head), skins, and weapon models
- Parse `animation.cfg` files to populate animation frame data
- Drive per-frame animation state machines for legs and torso (idle, jump, land, attack, drop/raise weapon)
- Compute hierarchical entity positioning via tag attachment (torso→legs, head→torso, weapon→torso, barrel→weapon)
- Calculate smooth angle transitions (yaw swing, pitch) for the displayed model
- Issue renderer calls to assemble and submit the full player scene each UI frame
- Manage weapon switching sequencing with audio cue

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `ui_shared.h`
- **Defined elsewhere:** `uiInfo` (global UI context, provides `uiDC.frameTime`); `bg_itemlist` (item/weapon definitions from `bg_misc.c`); all `trap_R_*`, `trap_CM_LerpTag`, `trap_S_*`, `trap_FS_*` syscall wrappers; math utilities (`AnglesToAxis`, `MatrixMultiply`, `AngleSubtract`, `AngleMod`, etc.); animation enum constants (`LEGS_JUMP`, `TORSO_ATTACK`, etc.)

# code/ui/ui_public.h
## File Purpose
Defines the public ABI contract between the Quake III Arena engine and the UI dynamic module (VM). It enumerates all syscall trap numbers the UI module uses to call into the engine (`uiImport_t`) and all entry points the engine calls on the UI module (`uiExport_t`).

## Core Responsibilities
- Declare the UI API version constant for compatibility checks
- Define `uiClientState_t` to carry connection/server state to the UI
- Enumerate all engine→UI import syscalls (`uiImport_t`)
- Enumerate all UI→engine export entry points (`uiExport_t`)
- Define `uiMenuCommand_t` for identifying which menu to activate
- Define server-list sort-order constants

## External Dependencies
- `connstate_t` — defined in engine connection-state headers (e.g., `client.h`)
- `MAX_STRING_CHARS` — defined in `q_shared.h`
- No includes are present in this header; consumers must include prerequisite headers before this file

# code/ui/ui_shared.c
## File Purpose
This is the shared UI framework implementation for Quake III Arena, providing the complete runtime for a data-driven menu system. It handles menu/item parsing from script files, rendering of all widget types, input routing (mouse, keyboard, key binding), and memory management for UI resources. It is shared between the `ui` and `cgame` modules via conditional compilation.

## Core Responsibilities
- Fixed-pool memory allocation and interned string storage for UI data
- Parsing menu/item definitions from PC (parser context) token streams using keyword hash tables
- Painting all window and item types (text, listbox, slider, model, bind, ownerdraw, etc.)
- Routing keyboard and mouse input to the focused menu/item
- Managing key bindings (read, write, defaults) via the `g_bindings` table
- Scripting: tokenizing and dispatching `commandList` scripts attached to items/menus
- Managing menu focus stack, visibility, transitions, orbiting, and fade effects

## External Dependencies
- `ui_shared.h` → `q_shared.h`, `tr_types.h`, `keycodes.h`, `menudef.h`
- `trap_PC_ReadToken`, `trap_PC_SourceFileAndLine`, `trap_PC_LoadSource` — defined in platform-specific syscall stubs
- `COM_ParseExt`, `Q_stricmp`, `Q_strcat`, `Q_strupr` — defined in `q_shared.c`
- `Com_Printf` — engine print, defined elsewhere
- `AxisClear`, `AnglesToAxis`, `VectorSet`, `VectorCopy` — math, defined in `q_math.c`
- All `DC->*` function pointers — resolved at runtime via `Init_Display`

# code/ui/ui_shared.h
## File Purpose
Defines the complete shared data model and public API for the Quake III Arena UI system, used by both the `ui` and `cgame` modules. It declares all menu/item/window types, the display context vtable, cached assets, and the full set of functions for menu lifecycle management and rendering.

## Core Responsibilities
- Define all UI structural types: `windowDef_t`, `itemDef_t`, `menuDef_t`, and their sub-types
- Declare the `displayContextDef_t` vtable that abstracts all renderer/engine calls away from UI code
- Declare `cachedAssets_t` for shared UI texture/font/sound handles
- Define window state flag bitmasks (WINDOW_*, CURSOR_*)
- Declare the full public API surface for menu/display management (init, paint, input, feeder, etc.)
- Define string pool constants and management API (`String_Alloc`, `String_Init`)
- Declare parser helpers for both text (`Float_Parse`, etc.) and PC (preprocessed script) token streams

## External Dependencies
- `../game/q_shared.h` — `vec4_t`, `qboolean`, `qhandle_t`, `sfxHandle_t`, `fontInfo_t`, `glconfig_t`, `refEntity_t`, `refdef_t`, `pc_token_t`
- `../cgame/tr_types.h` — `refEntity_t`, `refdef_t`, `glconfig_t`
- `keycodes.h` — `keyNum_t` enum
- `../../ui/menudef.h` — `ITEM_TYPE_*`, `FEEDER_*`, `CG_SHOW_*`, owner-draw constants
- `trap_PC_*` functions — defined elsewhere in the VM syscall table (`ui_syscalls.c` / `cg_syscalls.c`)
- `UI_Alloc` / `UI_InitMemory` / `UI_OutOfMemory` — VM-local memory pool, defined in `ui_main.c`
- `Controls_GetConfig` / `Controls_SetConfig` / `Controls_SetDefaults` — defined in `ui_shared.c`

# code/ui/ui_syscalls.c
## File Purpose
Provides the DLL-side system call bridge for the UI module, mapping high-level `trap_*` functions to indexed engine syscalls via a single function pointer. This file is only compiled for DLL builds; the QVM equivalent is `ui_syscalls.asm`.

## Core Responsibilities
- Store and initialize the engine-provided `syscall` function pointer via `dllEntry`
- Wrap every engine service (rendering, sound, cvars, filesystem, networking, input, cinematics) behind typed `trap_*` C functions
- Handle float-to-int reinterpretation via `PASSFLOAT` to safely pass floats through the variadic integer syscall ABI
- Expose CD-key validation and PunkBuster status reporting to the UI module
- Provide LAN/server browser query traps for the multiplayer server list UI

## External Dependencies
- `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`, `ui_shared.h`)
- `UI_*` syscall index constants — defined in `ui_public.h` (not in this file)
- All type definitions (`vmCvar_t`, `refEntity_t`, `glconfig_t`, `qtime_t`, `e_status`, `fontInfo_t`, etc.) — defined elsewhere in shared/game/renderer headers
- `QDECL` calling convention macro — defined in `q_shared.h`

# code/ui/ui_util.c
## File Purpose
A stub or placeholder utility file for the Quake III Arena UI module, intended to house memory and string allocation helpers for the new UI system. The file contains no implemented functions — only a license header and comment annotations.

## Core Responsibilities
- Reserved as the location for UI utility functions (memory, string allocation)
- No active responsibilities in current state; file is empty beyond the header

## External Dependencies
- None declared. No `#include` directives are present.
- Comment attributes (`origin: rad`) suggest authorship from the "rad" (RAD Game Tools / id internal) development context.

---

> **Note:** This file is effectively empty. The header comment indicates it was scaffolded to hold UI utility code (memory and string allocation), but no implementation was committed. Any actual utility functions intended for this file likely reside in `code/ui/ui_shared.c` or `code/ui/ui_atoms.c`.

# code/unix/linux_common.c
## File Purpose
Provides Linux/GAS-syntax x86 assembly implementations of `Com_Memcpy` and `Com_Memset` as drop-in replacements for the MSVC inline-asm versions in `qcommon/common.c`. The active code path (guarded by `#if 1`) simply delegates to libc `memcpy`/`memset`, while the disabled `#else` branch contains hand-optimized MMX/x86 assembly routines.

## Core Responsibilities
- Supply `Com_Memcpy` and `Com_Memset` as Linux platform overrides
- (Disabled) Implement a 32-byte-unrolled scalar x86 `memcpy` with alignment handling
- (Disabled) Implement an MMX-accelerated `memset` via `_copyDWord` for blocks ≥ 8 bytes
- (Disabled) Implement a software prefetch routine `Com_Prefetch` for read/read-write access patterns
- Convert MSVC `__asm` syntax to GAS `__asm__ __volatile__` with local labels and input/output constraints

## External Dependencies
- `<unistd.h>` — for `size_t`
- `<string.h>` — for `memcpy`, `memset` (active path)
- `Com_Prefetch` declared locally (disabled path); defined in same `#else` block
- `_copyDWord` defined locally (disabled path only)
- `Com_Memcpy` / `Com_Memset` symbols consumed by `qcommon/common.c` and the rest of the engine ("defined here, used everywhere")

# code/unix/linux_glimp.c
## File Purpose
This file implements all Linux/X11-specific OpenGL display initialization, input handling, and SMP render-thread support for Quake III Arena. It provides the platform-specific `GLimp_*` and `IN_*` entry points that the renderer and client layers depend on. It manages the X11 display connection, GLX context, video mode switching, mouse/keyboard grabbing, and gamma control.

## Core Responsibilities
- Create and manage the X11 window and GLX rendering context (`GLW_SetMode`)
- Load the OpenGL shared library and initialize GL extensions (`GLW_LoadOpenGL`, `GLW_InitExtensions`)
- Handle X11 events: keyboard, mouse (relative/DGA), buttons, window changes (`HandleEvents`)
- Grab/ungrab mouse and keyboard for in-game input (`install_grabs`, `uninstall_grabs`)
- Set display gamma via XF86VidMode extension (`GLimp_SetGamma`, `GLW_InitGamma`)
- Swap front/back buffers each frame (`GLimp_EndFrame`)
- Optionally spawn a dedicated render thread using pthreads (`GLimp_SpawnRenderThread` and SMP helpers)
- Initialize and shut down the input subsystem (`IN_Init`, `IN_Shutdown`, `IN_Frame`)

## External Dependencies
- **X11:** `<X11/Xlib.h>` (via GLX), `<X11/keysym.h>`, `<X11/cursorfont.h>`
- **XFree86 extensions:** `<X11/extensions/xf86dga.h>`, `<X11/extensions/xf86vmode.h>`
- **GLX:** `<GL/glx.h>`
- **pthreads:** `<pthread.h>`, `<semaphore.h>`
- **Dynamic linking:** `<dlfcn.h>` — `dlsym` used to resolve ARB extension function pointers from `glw_state.OpenGLLib`
- **Defined elsewhere:** `Sys_QueEvent`, `Sys_XTimeToSysTime`, `Sys_Milliseconds`, `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging`, `InitSig`, `IN_StartupJoystick`, `IN_JoyMove`, `glConfig`, `glState`, `ri` (refimport), `cls`, `com_developer`, all `r_*` cvars, all `q_*` string utilities.

# code/unix/linux_joystick.c
## File Purpose
Implements Linux-specific joystick input handling for Quake III Arena, translating Linux kernel joystick events (`/dev/jsN`) into the engine's internal key event system. It bridges the Linux joystick driver's event model to Quake's polling-style input pipeline.

## Core Responsibilities
- Open and initialize the first available joystick device (`/dev/js0`–`/dev/js3`)
- Drain the joystick event queue each frame
- Dispatch button press/release events directly as `SE_KEY` events
- Convert axis values to a bitmask and synthesize key press/release events for axis transitions
- Map 16 axes to virtual key codes (`joy_keys[]`)

## External Dependencies
- `<linux/joystick.h>` — `struct js_event`, `JS_EVENT_BUTTON`, `JS_EVENT_AXIS`, `JS_EVENT_INIT`, `JSIOCG*` ioctls
- `<fcntl.h>`, `<sys/ioctl.h>`, `<unistd.h>`, `<sys/types.h>` — POSIX I/O
- `../client/client.h` — `cvar_t`, `Com_Printf`, key code constants (`K_LEFTARROW`, `K_JOY1`, etc.)
- `linux_local.h` — `Sys_QueEvent`, `sysEventType_t` (`SE_KEY`)
- `Sys_QueEvent` — defined in `unix_main.c` (not this file)
- `in_joystick`, `in_joystickDebug`, `joy_threshold` — defined/registered in `linux_glimp.c`

# code/unix/linux_local.h
## File Purpose
Platform-specific header for the Linux port of Quake III Arena, declaring all Linux/Unix-specific subsystem interfaces. It serves as the internal contract between the Linux platform layer modules (input, GL, signals, system events).

## Core Responsibilities
- Declare system event queue injection interface
- Declare input subsystem lifecycle and per-frame functions
- Declare joystick startup and polling functions
- Declare OpenGL dynamic library (QGL) management interface
- Declare signal handler initialization
- Provide a `strlwr` utility absent from standard POSIX libc

## External Dependencies
- `q_shared.h` / `qcommon.h` — `qboolean`, `sysEventType_t`, `netadr_t`, `msg_t` (defined elsewhere)
- `dlopen`/`dlclose` — used by `QGL_Init`/`QGL_Shutdown` implementations (glibc `<dlfcn.h>`)
- POSIX `<signal.h>` — used by `InitSig` implementation

# code/unix/linux_qgl.c
## File Purpose
Implements the Linux/Unix operating system binding of OpenGL to QGL function pointers by dynamically loading an OpenGL shared library via `dlopen`/`dlsym`. It provides a thin indirection layer with two modes: direct dispatch (pointers point straight to the loaded library symbols) and logging dispatch (pointers point to `log*` wrappers that write to a file before forwarding to the real function).

## Core Responsibilities
- Load the OpenGL shared library at runtime using `dlopen`
- Resolve all ~230 OpenGL 1.1 entry points plus GLX and optional extension functions via `dlsym` (macro `GPA`)
- Expose the resolved addresses through the global `qgl*` function pointer table consumed by the rest of the renderer
- Maintain a parallel `dll*` shadow table holding the raw library addresses
- Provide per-call GL logging (writes function name/args to `gl.log`) by swapping `qgl*` pointers to `log*` wrappers
- Null out all `qgl*` pointers and close the library handle on shutdown

## External Dependencies
- `<dlfcn.h>` — `dlopen`, `dlclose`, `dlsym`, `dlerror`
- `<unistd.h>` — `getcwd`, `getuid`
- `../renderer/tr_local.h` — renderer globals (`r_logFile`, `ri`, `glw_state` usage context), `qboolean`, `Q_strcat`, `Com_sprintf`, `ri.Printf`, `ri.Cvar_*`
- `unix_glw.h` — `glwstate_t`, `glw_state` declaration
- `saved_euid` — defined in `code/unix/unix_main.c`; used to detect setuid execution and conditionally try CWD library lookup
- All `qgl*` function pointer declarations consumed by `code/renderer/` subsystem (defined elsewhere, populated here)

# code/unix/linux_signals.c
## File Purpose
Installs POSIX signal handlers for the Linux build of Quake III Arena, enabling graceful shutdown on fatal or termination signals. It guards against double-signal re-entry and optionally shuts down the OpenGL renderer before exiting.

## Core Responsibilities
- Register a unified `signal_handler` for all critical POSIX signals via `InitSig`
- Detect double-signal re-entry using a static flag and force-exit in that case
- Shut down the OpenGL/renderer subsystem (`GLimp_Shutdown`) on the first signal (non-dedicated build only)
- Delegate final process exit to `Sys_Exit` rather than calling `exit()` directly

## External Dependencies
- `<signal.h>` — POSIX signal API (`signal`, `SIGHUP`, `SIGQUIT`, etc.)
- `../game/q_shared.h` — `qboolean`, `qfalse`, `qtrue`
- `../qcommon/qcommon.h` — (included for shared definitions; no direct calls visible here)
- `../renderer/tr_local.h` — `GLimp_Shutdown` (included only when `DEDICATED` is not defined)
- `Sys_Exit` — declared via forward declaration (`void Sys_Exit(int)`); defined in `unix_main.c`
- `GLimp_Shutdown` — defined in `linux_glimp.c`

# code/unix/linux_snd.c
## File Purpose
Linux/FreeBSD platform-specific DMA sound driver for Quake III Arena. It opens the OSS `/dev/dsp` device, configures it for mmap-based DMA audio output, and implements the `SNDDMA_*` interface consumed by the portable sound mixing layer.

## Core Responsibilities
- Register and validate sound CVARs (`sndbits`, `sndspeed`, `sndchannels`, `snddevice`)
- Open the OSS sound device with privilege escalation (`seteuid`)
- Negotiate sample format, rate, and channel count via `ioctl`
- Memory-map the DMA ring buffer into `dma.buffer`
- Arm the DSP trigger to begin output
- Query the current playback pointer (`GETOPTR`) each frame
- Work around a glibc `memset` bug via a custom `Snd_Memset` fallback

## External Dependencies
- **System headers:** `<unistd.h>`, `<fcntl.h>`, `<sys/ioctl.h>`, `<sys/mman.h>`, `<linux/soundcard.h>` (Linux) / `<sys/soundcard.h>` (FreeBSD)
- **Local headers:** `../game/q_shared.h`, `../client/snd_local.h`
- **Defined elsewhere:**
  - `dma` (`dma_t`, global) — `snd_dma.c`
  - `saved_euid` (`uid_t`) — `unix_main.c`
  - `Cvar_Get`, `Com_Printf`, `Com_Memset` — engine core

# code/unix/qasm.h
## File Purpose
A shared header file for x86 assembly (`.s`/`.nasm`) translation units on Unix/Linux, providing C-to-assembly symbol name mangling, architecture detection macros, `.extern` declarations for all software-renderer and audio globals, and byte-offset constants for key C structs used directly from assembly code.

## Core Responsibilities
- Define the `C(label)` macro to handle ELF vs non-ELF symbol name decoration (`_` prefix)
- Detect x86 architecture and set `id386` accordingly
- Declare `.extern` references to all software-renderer globals (z-buffer, texture, lighting, span, edge, surface state) for use in `.s` assembly files
- Declare `.extern` references to audio mixer globals (`paintbuffer`, `snd_p`, etc.)
- Define byte-offset constants for C structs (`plane_t`, `hull_t`, `channel_t`, `edge_t`, `surf_t`, etc.) so assembly can perform field-access without the C type system
- Mirror C struct layouts precisely; comments throughout warn that offsets must stay in sync with their C counterparts

## External Dependencies
- No `#include` directives — entirely self-contained preprocessor/assembler definitions.
- Depends implicitly on the following C headers staying in sync (noted in comments):
  - `model.h` — `plane_t`, `hull_t`, `medge_t`, `mvertex_t`, `mtriangle_t`, `dnode_t`
  - `sound.h` — `sfxcache_t`, `channel_t`, `portable_samplepair_t`
  - `r_shared.h` — `espan_t`, `edge_t`, `surf_t`
  - `d_local.h` — `sspan_t`
  - `d_polyset.c` — `spanpackage_t`
  - `r_local.h` — `clipplane_t`, `NEAR_CLIP`, `CYCLE`
  - `render.h` — `refdef_t`
- External symbols used but defined elsewhere (selected significant ones):

| Symbol | Likely Owner |
|---|---|
| `d_pzbuffer`, `d_zistepu`, `d_ziorigin` | Software renderer depth/z subsystem |
| `paintbuffer`, `snd_p`, `snd_out`, `snd_vol` | Audio mixer (`snd_mix.c`) |
| `r_turb_*` | Turbulent surface rasterizer |
| `edge_p`, `surface_p`, `surfaces`, `span_p` | Renderer edge/surface list manager |
| `aliastransform`, `r_avertexnormals` | Alias model renderer |
| `D_PolysetSetEdgeTable`, `D_RasterizeAliasPolySmooth` | Polyset rasterizer (C entry points called from ASM) |
| `vright`, `vup`, `vpn` | View orientation vectors |

# code/unix/unix_glw.h
## File Purpose
Declares the platform-specific OpenGL window state structure for Linux/FreeBSD. It defines a single shared state object used by the Unix OpenGL window and rendering subsystem.

## Core Responsibilities
- Guards inclusion to Linux/FreeBSD platforms only via a compile-time `#error` directive
- Defines the `glwstate_t` struct holding Unix GL window state
- Exposes `glw_state` as an `extern` global for use across the Unix GL subsystem

## External Dependencies
- `<stdio.h>` — implied by `FILE *log_fp` (must be included before this header by consumers)
- `linux_glimp.c` — defines `glw_state` (definition lives elsewhere)
- No Quake-specific headers; this file is intentionally minimal and low-level

# code/unix/unix_main.c
## File Purpose
This is the Linux/Unix platform entry point for Quake III Arena, implementing the OS-level system layer. It provides the `main()` function, the event loop, DLL loading, TTY console I/O, and all `Sys_*` functions required by the engine's platform abstraction.

## Core Responsibilities
- Houses `main()`: parses args, initializes engine via `Com_Init`, and runs the main `Com_Frame` loop
- Implements the system event queue (`Sys_QueEvent` / `Sys_GetEvent`) feeding input, console, and network events
- Provides TTY console with raw-mode input, line editing, tab completion, and command history
- Implements `Sys_LoadDll` / `Sys_UnloadDll` for native game/cgame/ui module loading via `dlopen`
- Implements `Sys_Error`, `Sys_Quit`, `Sys_Exit` — the unified shutdown/error paths
- Provides no-op or pass-through background file streaming stubs
- Configures architecture cvar and FPU state at startup

## External Dependencies
- **Includes:** `<dlfcn.h>`, `<termios.h>`, `<sys/time.h>`, `<signal.h>`, `<mntent.h>` (Linux), `<fpu_control.h>` (Linux i386)
- **Defined elsewhere:** `Com_Init`, `Com_Frame`, `NET_Init`, `CL_Shutdown`, `IN_Init/Shutdown/Frame`, `Sys_SendKeyEvents`, `Sys_GetPacket`, `Sys_Milliseconds`, `FS_BuildOSPath`, `FS_Read`, `FS_Seek`, `Field_Clear`, `Field_CompleteCommand`, `Z_Malloc`, `Z_Free`, `Cvar_Get`, `Cvar_Set`, `Cvar_VariableString`, `Cmd_AddCommand`, `MSG_Init`, `InitSig`, `Sys_Cwd`, `Sys_SetDefaultCDPath`, `Sys_GetCurrentUser`

# code/unix/unix_net.c
## File Purpose
Implements the Unix/Linux (and macOS) platform-specific network layer for Quake III Arena, providing UDP socket creation, packet send/receive, local address enumeration, and LAN classification. It fulfills the `Sys_*` and `NET_*` network API required by the engine's platform-agnostic common layer (`qcommon`).

## Core Responsibilities
- Convert between engine `netadr_t` and POSIX `sockaddr_in` representations
- Open and close UDP sockets for IP (and stub IPX) communication
- Send and receive raw UDP packets
- Enumerate the host's local IP addresses (platform-divergent: Mac vs. generic POSIX)
- Classify an address as LAN or WAN (RFC 1918 class A/B/C awareness)
- Provide a blocking/sleeping select-based idle for dedicated server frame throttling

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `byte`, `netadr_t`, `cvar_t`, `Com_Printf`, `Com_Error`, `Q_stricmp`, `Com_sprintf`
- `../qcommon/qcommon.h` — `msg_t`, `netadrtype_t`, `NET_AdrToString`, `Cvar_Get`, `Cvar_SetValue`, `PORT_SERVER`, `com_dedicated`
- POSIX headers: `<sys/socket.h>`, `<netinet/in.h>`, `<netdb.h>`, `<arpa/inet.h>`, `<sys/ioctl.h>`, `<errno.h>`
- macOS-only: `<sys/sockio.h>`, `<net/if.h>`, `<net/if_dl.h>`, `<net/if_types.h>`
- **Defined elsewhere:** `NET_AdrToString`, `com_dedicated`, `stdin_active`

# code/unix/unix_shared.c
## File Purpose
Provides Unix/Linux platform-specific system utility functions shared across the engine — timing, filesystem enumeration, path resolution, and miscellaneous CPU/user queries. It implements the `Sys_*` interface declared in `qcommon.h` for POSIX-compliant platforms.

## Core Responsibilities
- High-resolution millisecond timer via `gettimeofday`
- Sub-frame X11 event timing correction (Linux non-dedicated only)
- Directory creation (`Sys_Mkdir`)
- Recursive and filtered file listing (`Sys_ListFiles`, `Sys_ListFilteredFiles`)
- Platform path resolution: CD path, install path, home path
- Current user and processor count queries
- Optional PPC/Apple `Sys_SnapVector` / `fastftol` fallbacks

## External Dependencies
- **Includes:** `<sys/types.h>`, `<sys/stat.h>`, `<errno.h>`, `<stdio.h>`, `<dirent.h>`, `<unistd.h>`, `<sys/mman.h>`, `<sys/time.h>`, `<pwd.h>`
- **Local headers:** `../game/q_shared.h`, `../qcommon/qcommon.h`
- **Defined elsewhere:** `CopyString`, `Z_Malloc`, `Z_Free`, `Com_sprintf`, `Com_FilterPath`, `Q_stricmp`, `Q_strncpyz`, `Q_strcat`, `Sys_Error`, `Com_Printf`; `cvar_t *in_subframe` (declared `extern`, defined in Linux input code)

# code/unix/vm_x86.c
## File Purpose
This is the Linux/Unix x86-specific stub for the Quake III Virtual Machine (Q3VM) JIT compiler. It provides empty placeholder implementations of `VM_Compile` and `VM_CallCompiled`, indicating the x86 JIT backend was not implemented (or not yet ported) for this Unix target.

## Core Responsibilities
- Satisfies the linker requirement for `VM_Compile` and `VM_CallCompiled` on Unix/x86 builds
- Acts as a no-op stub — the Unix build falls back to the interpreted VM path (`VM_CallInterpreted`) rather than JIT-compiled execution
- Mirrors the interface contract declared in `vm_local.h`

## External Dependencies

- **`../qcommon/vm_local.h`** — brings in `vm_t`, `vmHeader_t`, `opcode_t`, `vmSymbol_t`, and the full Q3VM interface declarations
- **`../game/q_shared.h`** (transitively) — base types (`qboolean`, `byte`, `MAX_QPATH`, etc.)
- **`qcommon.h`** (transitively) — `vmHeader_t` definition and common engine declarations
- **Defined elsewhere:** `VM_PrepareInterpreter`, `VM_CallInterpreted`, `currentVM`, `vm_debugLevel` — all implemented in `qcommon/vm_interpreted.c` and `qcommon/vm.c`

# code/win32/glw_win.h
## File Purpose
Declares the Win32-specific OpenGL window state structure (`glwstate_t`) and its global instance. It encapsulates all Win32/WGL handles and display configuration needed to manage the OpenGL rendering context on Windows.

## Core Responsibilities
- Define the `glwstate_t` struct holding all Win32 GL window state
- Store Win32 handles: device context (HDC), GL rendering context (HGLRC), OpenGL DLL instance
- Track desktop display properties (bit depth, resolution)
- Track fullscreen mode and pixel format initialization state
- Expose the global `glw_state` instance to other translation units
- Guard against inclusion on non-Win32 platforms via `#error`

## External Dependencies
- `<windows.h>` (implicit) — provides `WNDPROC`, `HDC`, `HGLRC`, `HINSTANCE`, `FILE`
- `qboolean` — defined in `q_shared.h` (engine-wide boolean typedef)
- `glwstate_t glw_state` — defined externally in `code/win32/win_glimp.c`

# code/win32/resource.h
## File Purpose
Auto-generated Windows resource identifier header for the Quake III Arena Win32 build. It defines numeric IDs for embedded Win32 resources (icons, bitmaps, cursors, strings) referenced by `winquake.rc`.

## Core Responsibilities
- Define symbolic integer constants for Win32 resource IDs (icons, bitmaps, cursors, string tables)
- Provide APSTUDIO bookkeeping macros so Visual Studio's resource editor knows the next available ID values for each resource category
- Act as the bridge between the `.rc` resource script and C/C++ source code that references resources by name

## External Dependencies
- **Consumed by:** `code/win32/winquake.rc` (resource script referencing these IDs)
- **Potentially referenced by:** Win32 platform code in `code/win32/` that loads icons, cursors, or bitmaps via `LoadIcon`, `LoadCursor`, `LoadBitmap`, etc.
- No standard library includes; no external symbols are used or defined here.

| Resource Constant | Value | Kind |
|---|---|---|
| `IDS_STRING1` | 1 | String table entry |
| `IDI_ICON1` | 1 | Icon resource |
| `IDB_BITMAP1` | 1 | Bitmap resource |
| `IDB_BITMAP2` | 128 | Bitmap resource |
| `IDC_CURSOR1` | 129 | Cursor resource |
| `IDC_CURSOR2` | 130 | Cursor resource |
| `IDC_CURSOR3` | 131 | Cursor resource |

# code/win32/win_gamma.c
## File Purpose
Manages hardware gamma ramp correction on Win32, using either the 3Dfx-specific WGL extension or the standard Win32 `SetDeviceGammaRamp` API. It saves the original gamma on init, applies game-specified gamma tables per frame, and restores the original on shutdown.

## Core Responsibilities
- Detect whether the hardware/driver supports gamma ramp modification (`WG_CheckHardwareGamma`)
- Save the pre-game hardware gamma ramp for later restoration
- Validate saved gamma ramp sanity (monotonically increasing, crash-recovery linear fallback)
- Apply per-channel RGB gamma ramp tables to the display device (`GLimp_SetGamma`)
- Apply Windows 2000-specific gamma clamping restrictions
- Enforce monotonically increasing gamma values before submission
- Restore original hardware gamma on game exit (`WG_RestoreGamma`)

## External Dependencies
- `<assert.h>` — standard C (unused in active code paths)
- `../renderer/tr_local.h` — `glConfig` (`glconfig_t`), `ri` (`refimport_t`), `r_ignorehwgamma` cvar
- `../qcommon/qcommon.h` — `Com_DPrintf`, `Com_Printf`
- `glw_win.h` — `glw_state` (`glwstate_t`), exposes `glw_state.hDC`
- `win_local.h` — Win32 headers (`windows.h`), `OSVERSIONINFO`, `GetVersionEx`
- `qwglSetDeviceGammaRamp3DFX`, `qwglGetDeviceGammaRamp3DFX` — defined elsewhere (WGL extension pointers, loaded in `win_glimp.c`)
- `glConfig.deviceSupportsGamma`, `glConfig.driverType` — defined in renderer globals (`tr_init.c`)

# code/win32/win_glimp.c
## File Purpose
Win32-specific OpenGL initialization, window management, and frame presentation layer for Quake III Arena. It implements the platform-facing `GLimp_*` interface required by the renderer, handling everything from pixel format selection and WGL context creation to fullscreen CDS mode switching and optional SMP render thread synchronization.

## Core Responsibilities
- Create and destroy the Win32 application window (`HWND`)
- Select an appropriate `PIXELFORMATDESCRIPTOR` and establish a WGL rendering context
- Handle fullscreen mode switching via `ChangeDisplaySettings` (CDS)
- Load an OpenGL DLL (ICD, standalone, or Voodoo) and bind all function pointers via `QGL_Init`
- Probe and enable supported OpenGL/WGL extensions (multitexture, S3TC, swap control, 3DFX gamma, CVA)
- Perform per-frame buffer swap and swap-interval management in `GLimp_EndFrame`
- Provide SMP support: spawn a render thread and coordinate it with event objects

## External Dependencies
- `../renderer/tr_local.h` — `glConfig`, `glState`, `ri` (refimport), renderer cvars
- `../qcommon/qcommon.h` — `cvar_t`, `ri.Cvar_Get`, `ri.Error`, `ri.Printf`
- `glw_win.h` — `glwstate_t` definition
- `win_local.h` — `WinVars_t g_wv` (hWnd, hInstance), Win32 headers
- `resource.h` — `IDI_ICON1` icon resource
- **Defined elsewhere:** `QGL_Init`, `QGL_Shutdown`, `QGL_EnableLogging` (`win_qgl.c`); `WG_CheckHardwareGamma`, `WG_RestoreGamma` (`win_gamma.c`); `R_GetModeInfo` (renderer); all `qwgl*`/`qgl*` function pointers (QGL layer); `g_wv` (`win_main.c`).

# code/win32/win_input.c
## File Purpose
Win32-specific input handling for Quake III Arena, managing mouse (both Win32 raw and DirectInput), joystick, and MIDI controller input. It translates hardware input events into engine-queued system events via `Sys_QueEvent`.

## Core Responsibilities
- Initialize, activate, deactivate, and shut down Win32 mouse and DirectInput mouse
- Poll DirectInput buffered mouse data and queue button/wheel/motion events
- Initialize and poll Win32 Multimedia joystick API, mapping axes and buttons to key events
- Initialize and receive MIDI input, mapping MIDI notes to aux key events
- Per-frame input dispatch (`IN_Frame`), including delayed DirectInput init fallback
- Register input-related cvars (`in_mouse`, `in_joystick`, `in_midi`, etc.)

## External Dependencies
- `../client/client.h` — `cls` (keyCatchers), `Cvar_*`, `Cmd_*`, `Com_Printf`, `Sys_QueEvent`, key constants
- `win_local.h` — `g_wv` (hWnd, hInstance, osversion, sysMsgTime), Win32/DInput/DSound headers
- Win32 APIs: `dinput.dll` (loaded dynamically), `winmm` (joystick/MIDI via `joyGetPosEx`, `midiInOpen`)
- Defined elsewhere: `Sys_QueEvent`, `Cvar_Set/Get/VariableValue`, `Com_Printf`, `Com_Memset`, `g_wv`, `cls`

# code/win32/win_local.h
## File Purpose
Win32-platform-specific header for Quake III Arena, declaring the shared Windows application state, input/sound subsystem interfaces, and window procedure used across all Win32 platform modules.

## Core Responsibilities
- Declares the `WinVars_t` struct holding global Win32 application state (window handle, instance, OS version, etc.)
- Declares the input subsystem API (`IN_*` functions)
- Declares the system event queue injection point (`Sys_QueEvent`)
- Declares the Win32 console management functions
- Declares the DirectSound activation and init hooks
- Exports the main window procedure (`MainWndProc`)

## External Dependencies
- `<windows.h>` — Win32 API types (`HWND`, `HINSTANCE`, `OSVERSIONINFO`, `LONG`, etc.)
- `<dinput.h>` — DirectInput 3.0 (input device enumeration/polling)
- `<dsound.h>` — DirectSound 3.0 (audio output)
- `<winsock.h>` / `<wsipx.h>` — Winsock + IPX networking
- `sysEventType_t`, `netadr_t`, `msg_t`, `usercmd_t`, `qboolean` — defined in `qcommon.h` / `q_shared.h` (engine shared headers, included transitively by including modules)
- `g_wv` — defined in `code/win32/win_main.c`

# code/win32/win_main.c
## File Purpose
Win32-specific platform entry point and system abstraction layer for Quake III Arena. It implements `WinMain`, the OS event loop, file system enumeration, DLL loading, and miscellaneous system services required by the engine's `Sys_*` API contract.

## Core Responsibilities
- Owns `WinMain` and the top-level game loop (`IN_Frame` → `Com_Frame`)
- Queues and dispatches system events (keyboard, mouse, network, console) via `Sys_QueEvent` / `Sys_GetEvent`
- Provides Win32 filesystem services: directory listing, filtered file enumeration, `Sys_Mkdir`, `Sys_Cwd`
- Loads and unloads game module DLLs (`Sys_LoadDll` / `Sys_UnloadDll`) with a security warning in release builds
- Implements stub background file streaming (`Sys_InitStreamThread` etc.) — full threaded implementation is `#if 0`'d out
- Initializes the system: high-resolution timer, OS version detection, CPU identification, input subsystem
- Handles fatal error and clean-quit paths (`Sys_Error`, `Sys_Quit`)

## External Dependencies

- `../client/client.h` — `IN_Frame`, `IN_Init`, `IN_Shutdown`
- `../qcommon/qcommon.h` — `Com_Init`, `Com_Frame`, `NET_Init`, `NET_Restart`, `Sys_Milliseconds`, `Z_Malloc`, `Z_Free`, `FS_Read`, `FS_Seek`, `Cvar_*`, `Cmd_AddCommand`, `MSG_Init`
- `win_local.h` — `WinVars_t g_wv`, `Sys_CreateConsole`, `Sys_DestroyConsole`, `Sys_ConsoleInput`, `Sys_GetPacket`, `MainWndProc`, `Conbuf_AppendText`, `Sys_ShowConsole`, `Sys_SetErrorText`
- Win32 API: `<windows.h>`, `timeBeginPeriod`/`timeEndPeriod`/`timeGetTime` (winmm), `LoadLibrary`, `GetProcAddress`, `FreeLibrary`, `GlobalMemoryStatus`, `_findfirst`/`_findnext`/`_findclose`
- **Defined elsewhere:** `FS_BuildOSPath`, `Sys_GetProcessorId`, `Sys_GetCurrentUser`, `Sys_Milliseconds`, `CopyString`, `Q_strncpyz`, `Com_sprintf`, `Com_FilterPath`

# code/win32/win_net.c
## File Purpose
Windows-specific (Winsock) implementation of the low-level network layer for Quake III Arena. It creates and manages UDP sockets for IP and IPX protocols, handles SOCKS5 proxy tunneling, and provides packet send/receive primitives consumed by the platform-independent `qcommon` network layer.

## Core Responsibilities
- Initialize and shut down the Winsock library (`WSAStartup`/`WSACleanup`)
- Open, configure, and close UDP sockets for IP (`ip_socket`) and IPX (`ipx_socket`) protocols
- Implement optional SOCKS5 proxy negotiation and UDP-associate relay
- Convert between engine `netadr_t` and OS `sockaddr`/`sockaddr_ipx` representations
- Receive incoming packets (`Sys_GetPacket`) and send outgoing packets (`Sys_SendPacket`)
- Classify remote addresses as LAN or WAN (`Sys_IsLANAddress`)
- Enumerate and cache local IP addresses for LAN detection

## External Dependencies
- `<winsock.h>`, `<wsipx.h>` — Winsock and IPX socket APIs (via `win_local.h`)
- `../game/q_shared.h` — `qboolean`, `byte`, `cvar_t`, `netadr_t` type definitions
- `../qcommon/qcommon.h` — `msg_t`, `NET_AdrToString`, `Com_Printf`, `Com_Error`, `Cvar_Get`, `Cvar_SetValue`, `PORT_ANY`, `PORT_SERVER`, `NA_*` address type constants
- `NET_AdrToString` — defined in `qcommon/net_chan.c`, not in this file
- `NET_SendPacket` (higher-level wrapper) — defined in `qcommon/net_chan.c`

# code/win32/win_qgl.c
## File Purpose
Windows-specific binding layer that dynamically loads `opengl32.dll` (or a 3Dfx Glide wrapper) and assigns all OpenGL 1.x and WGL function pointers to the engine's `qgl*`/`qwgl*` indirection layer. It also implements an optional per-call logging path that intercepts every GL call and writes a human-readable trace to a log file.

## Core Responsibilities
- Load an OpenGL DLL via `LoadLibrary` and resolve all `gl*`/`wgl*` symbols via `GetProcAddress` (`QGL_Init`)
- Null-out and free the DLL handle on shutdown (`QGL_Shutdown`)
- Maintain two parallel function-pointer sets: `dll*` (direct DLL pointers) and `qgl*`/`qwgl*` (active pointers used by the renderer)
- Swap active pointers between direct (`dll*`) and logging (`log*`) wrappers on demand (`QGL_EnableLogging`)
- Emit per-call human-readable GL traces to a timestamped `gl.log` file when logging is enabled
- Validate 3Dfx Glide availability before loading the 3Dfx driver

## External Dependencies
- `#include <float.h>` — standard C
- `#include "../renderer/tr_local.h"` — provides `ri` (refimport), `r_logFile` cvar, `glconfig_t`, renderer types
- `#include "glw_win.h"` — provides `glwstate_t` and `glw_state` (the Win32 GL window/context state)
- **Defined elsewhere:** `glw_state` (defined in `win_glimp.c`); `ri` (renderer import table); `r_logFile`, `qglActiveTextureARB`, `qglClientActiveTextureARB`, `qglMultiTexCoord2fARB`, `qglLockArraysEXT`, `qglUnlockArraysEXT` (declared/used in renderer modules); Windows API: `LoadLibrary`, `FreeLibrary`, `GetProcAddress`, `GetSystemDirectory`

# code/win32/win_shared.c
## File Purpose
Provides Windows-specific implementations of shared system services required by the Quake III engine, including timing, floating-point snapping, CPU feature detection, and user/path queries. This file bridges the platform-agnostic `Sys_*` interface declared in `qcommon.h` to Win32 APIs.

## Core Responsibilities
- Provide `Sys_Milliseconds` using `timeGetTime()` with a stable epoch base
- Implement `Sys_SnapVector` via x86 FPU inline assembly (`fistp`) for fast float-to-int truncation
- Detect CPU capabilities (Pentium, MMX, 3DNow!, KNI/SSE) via CPUID and return a capability constant
- Query the Windows username via `GetUserName`
- Provide default home/install path resolution

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `qtrue`/`qfalse`, shared types
- `../qcommon/qcommon.h` — `CPUID_*` constants, `Sys_*` declarations, `Sys_Cwd`
- `win_local.h` — `WinVars_t`, Win32 subsystem headers
- `<windows.h>` (via `win_local.h`) — `GetUserName`
- `<mmsystem.h>` (implicit via WinMM link) — `timeGetTime`
- `Sys_Cwd` — defined elsewhere (not in this file)

# code/win32/win_snd.c
## File Purpose
Windows-specific DirectSound DMA backend for Quake III Arena's audio system. It implements the platform sound device interface (`SNDDMA_*`) using DirectSound COM APIs to drive a looping secondary buffer that the portable mixer writes into.

## Core Responsibilities
- Initialize and tear down a DirectSound device via COM (`CoCreateInstance`)
- Create and configure a secondary DirectSound buffer (hardware-preferred, software fallback)
- Lock/unlock the circular DMA buffer each frame so the mixer can write samples
- Report the current playback position within the DMA ring buffer
- Re-establish the cooperative level when the application window changes focus

## External Dependencies
- `../client/snd_local.h` — `dma_t dma`, `channel_t`, `SNDDMA_*` declarations, `S_Shutdown`
- `win_local.h` — `WinVars_t g_wv` (for `hWnd`), DirectSound/DirectInput version defines, Win32 headers
- `<dsound.h>`, `<windows.h>` — DirectSound COM interfaces
- `Com_Printf`, `Com_DPrintf` — defined in `qcommon`
- `g_wv.hWnd` — window handle from the Win32 platform layer
- `S_Shutdown` — portable sound shutdown, defined in `client/snd_dma.c`

# code/win32/win_syscon.c
## File Purpose
Implements the Win32 system console window for Quake III Arena, providing a dedicated GUI console for the dedicated server and optional viewlog window for the client. It creates and manages a native Win32 popup window with a scrollable text buffer, command input line, and action buttons.

## Core Responsibilities
- Create and destroy the Win32 console popup window (`Sys_CreateConsole` / `Sys_DestroyConsole`)
- Handle Win32 window messages for the console and input-line subclassed control
- Append formatted output text to the scrollable edit buffer (`Conbuf_AppendText`)
- Poll and return text typed in the console input line (`Sys_ConsoleInput`)
- Show, hide, or minimize the console based on `visLevel` (`Sys_ShowConsole`)
- Display a flashing error banner when a fatal error occurs (`Sys_SetErrorText`)
- Relay quit/close commands from the console window back into the engine event queue

## External Dependencies
- `../client/client.h` — pulls in `q_shared.h`, `qcommon.h`, cvar types, `Sys_Error`, `Sys_Print`, `Sys_QueEvent`, `CopyString`, `Cvar_Set`, `Q_IsColorString`, `Q_strncpyz`, `va`
- `win_local.h` — `WinVars_t g_wv` (hInstance), `Sys_QueEvent` declaration, Win32 headers (`windows.h`, DirectInput, DirectSound, WinSock)
- `resource.h` — `IDI_ICON1` resource ID
- `com_viewlog`, `com_dedicated` — cvars defined elsewhere in `qcommon`
- `SE_CONSOLE` — sysEventType enum value defined in `qcommon.h`

# code/win32/win_wndproc.c
## File Purpose
Implements the Win32 window procedure (`MainWndProc`) for Quake III Arena, translating Windows OS messages into engine input events and managing window lifecycle, focus, and Alt-Tab suppression.

## Core Responsibilities
- Translate Win32 keyboard/mouse messages into engine `SE_KEY`/`SE_CHAR` events via `Sys_QueEvent`
- Map Windows scan codes to Quake key numbers, disambiguating numpad vs. cursor keys
- Handle mouse wheel input for both legacy (MSH_MOUSEWHEEL) and modern (WM_MOUSEWHEEL) paths
- Manage application focus/activation state and mouse capture toggling
- Suppress Alt-Tab on NT vs. 9x using platform-specific Win32 APIs
- Handle window creation/destruction, position tracking, and quit on `WM_CLOSE`
- Toggle fullscreen mode on Alt+Enter (`WM_SYSKEYDOWN` + VK_Return)

## External Dependencies
- `../client/client.h` — `cls`, `KEYCATCH_CONSOLE`, `Key_ClearStates`, `Cbuf_*`, `Cvar_*`, `Com_DPrintf`
- `win_local.h` — `WinVars_t`, `g_wv`, `IN_Activate`, `IN_MouseEvent`, `Sys_QueEvent`, `SNDDMA_Activate`
- `<windows.h>`, `<dinput.h>` — Win32 message constants, `HWND`, `BOOL`, `RegisterHotKey`, `SystemParametersInfo`
- **Defined elsewhere:** `in_mouse`, `in_logitechbug` (extern cvars from `win_input.c`); all `K_*` key constants (from `keys.h`); `Sys_QueEvent` (from `win_main.c`)

