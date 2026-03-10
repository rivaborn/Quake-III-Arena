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

