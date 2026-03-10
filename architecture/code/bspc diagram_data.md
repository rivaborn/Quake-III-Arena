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

