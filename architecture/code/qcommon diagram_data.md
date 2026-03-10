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

