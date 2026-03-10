# common/aselib.c
## File Purpose
Parses Autodesk ASCII Scene Export (ASE) 3D model files used by Quake III Arena's toolchain. It loads mesh geometry, materials, and animation frame sequences from ASE text format into in-memory structures, then exposes them as `polyset_t` arrays for consumption by downstream tools (model compilers, map tools).

## Core Responsibilities
- Read and tokenize an ASE file from disk into a flat memory buffer
- Parse the ASE hierarchy: `MATERIAL_LIST` → `GEOMOBJECT` → `MESH` / `MESH_ANIMATION`
- Build per-frame mesh arrays (vertices, faces, texture vertices, texture faces)
- Resolve material bitmap paths relative to `gamedir`, normalizing path separators
- Filter/discard objects named "Bip", "ignore_", or improperly labeled bodies when grabbing animations
- Convert parsed mesh data into `polyset_t` / `triangle_t` structures for external consumers
- Free all heap-allocated mesh frame data on request

## External Dependencies
- **`aselib.h`** → pulls in `cmdlib.h` (`Error`, `Q_filelength`, `gamedir`), `mathlib.h` (`qboolean`), `polyset.h` (`polyset_t`, `triangle_t`)
- `gamedir` — global string defined in `cmdlib`; used for material path resolution
- `Error` — fatal error handler defined in `cmdlib`
- `Q_filelength` — file size utility defined in `cmdlib`
- Standard C: `<stdio.h>`, `<stdlib.h>`, `<assert.h>`

# common/aselib.h
## File Purpose
Public API header for loading and querying 3D mesh data from ASE (ASCII Scene Export) files, a text-based format used by 3ds Max. It exposes an interface for the build tools (q3map, bspc) to import static and animated mesh geometry for use in map compilation and model processing.

## Core Responsibilities
- Declare the ASE file loader entry point
- Expose surface enumeration (count, name, animation frames)
- Declare the cleanup/free routine for loaded ASE data
- Pull in common tool-layer dependencies (`cmdlib`, `mathlib`, `polyset`)

## External Dependencies
- `common/cmdlib.h` — `qboolean`, file I/O utilities, error handling
- `common/mathlib.h` — `vec3_t`, `vec_t` used inside `triangle_t` via `polyset.h`
- `common/polyset.h` — `polyset_t`, `triangle_t` — the primary geometry output type
- Implementation (`aselib.c`) defined elsewhere; all state is opaque to callers

# common/bspfile.c
## File Purpose
Implements the BSP (Binary Space Partitioning) file I/O layer for Quake III Arena's offline tools (q3map, bspc, etc.). It owns all global BSP lump arrays, handles loading and writing BSP files with byte-order swapping, and provides entity key/value parsing utilities.

## Core Responsibilities
- Define and own all global BSP lump data arrays (geometry, visibility, lighting, entities, etc.)
- Load a BSP file from disk, copy each lump into its global array, and byte-swap all data
- Write global BSP arrays back to disk as a well-formed BSP file with a corrected header
- Byte-swap individual BSP structs for endian portability
- Parse the raw entity string (`dentdata`) into an in-memory `entity_t` array
- Serialize the `entity_t` array back into the `dentdata` string
- Provide key/value accessors (`ValueForKey`, `SetKeyValue`, `FloatForKey`, `GetVectorForKey`) for entity manipulation

## External Dependencies
- `cmdlib.h` — `LoadFile`, `SafeWrite`, `SafeOpenWrite`, `Error`, `copystring`, `LittleLong`, `LittleFloat`, `qboolean`
- `mathlib.h` — `vec_t`, `vec3_t`
- `bspfile.h` — BSP struct types (`dheader_t`, `dmodel_t`, `dleaf_t`, `drawVert_t`, `dsurface_t`, etc.), lump index constants, `MAX_MAP_*` limits (all defined via `qfiles.h` / `surfaceflags.h`)
- `scriplib.h` — `ParseFromMemory`, `GetToken`, `token` global (used by `ParseEntity`/`ParseEpair`)
- `GetLeafNums` — declared but never called in this file; defined elsewhere

# common/bspfile.h
## File Purpose
Declares the shared global BSP map data arrays and counts used by map compilation tools (q3map, bspc, q3radiant), as well as the higher-level `entity_t` / `epair_t` types and the API for loading, writing, and querying parsed BSP/entity data.

## Core Responsibilities
- Extern-declares all flat BSP lump arrays (geometry, visibility, lighting, shaders, etc.) shared across compilation tool translation units
- Provides the `entity_t` and `epair_t` types representing parsed map entities and their key/value metadata
- Declares the BSP file I/O entry points: `LoadBSPFile`, `WriteBSPFile`, `PrintBSPFileSizes`
- Declares the entity-string parse/unparse cycle (`ParseEntities` / `UnparseEntities`)
- Provides key/value query helpers (`ValueForKey`, `FloatForKey`, `GetVectorForKey`, `SetKeyValue`)
- Conditionally includes `qfiles.h` and `surfaceflags.h` from either the tool (`_TTIMOBUILD`) or engine path

## External Dependencies
- `qfiles.h` (via `_TTIMOBUILD` path or `../code/qcommon/qfiles.h`) — defines all `d*_t` BSP lump structs, `MAX_MAP_*` limits, `drawVert_t`, `dsurface_t`, `mapSurfaceType_t`
- `surfaceflags.h` (via `_TTIMOBUILD` path or `../code/game/surfaceflags.h`) — `CONTENTS_*` and `SURF_*` flag bit definitions
- `vec3_t`, `vec_t`, `byte` — defined in `q_shared.h` (pulled in transitively); not defined in this file
- `bspbrush_s`, `parseMesh_s` — forward-declared struct tags; defined in other map-compiler translation units

# common/cmdlib.c
## File Purpose
General-purpose utility library for Quake III's offline tools (q3map, bspc, q3radiant). Provides filesystem I/O, path manipulation, string utilities, argument parsing, byte-order conversion, and CRC computation — the shared foundation for all build/compile-time tools.

## Core Responsibilities
- File I/O: safe open/read/write, load/save whole files, file existence and length queries
- Path manipulation: qdir/gamedir resolution, expansion, stripping, extraction of parts
- String utilities: case-insensitive compare, upper/lower, token parser (`COM_Parse`)
- Command-line argument handling: `CheckParm`, wildcard expansion (Win32 only)
- Byte-order (endian) conversion for short, long, and float
- CCITT CRC-16 computation
- Directory creation and file archiving (`CreatePath`, `QCopyFile`)
- Verbose and broadcast-capable print wrappers (`_printf`, `qprintf`)

## External Dependencies
- `<sys/types.h>`, `<sys/stat.h>`, `<time.h>`, `<errno.h>`, `<stdarg.h>`, `<stdio.h>`, `<stdlib.h>`, `<string.h>`, `<ctype.h>`
- Win32: `<windows.h>`, `<direct.h>`, `<io.h>` (for `_findfirst`/`_findnext`, `_getcwd`, `_mkdir`, `FindWindow`, `PostMessage`, `GlobalAddAtom`)
- NeXT: `<libc.h>`
- `cmdlib.h` — declares all exported symbols and defines `qboolean`, `byte`, `MEM_BLOCKSIZE`
- `Q_getwd`, `QCopyFile`, `Q_mkdir` — defined in this file; no external symbols left undefined

# common/cmdlib.h
## File Purpose
A shared utility header for Quake III Arena's offline tools (q3map, bspc, q3radiant, q3asm). It declares a portable C runtime abstraction layer covering file I/O, string manipulation, path handling, endian conversion, argument parsing, and CRC computation used across all build-time tool executables.

## Core Responsibilities
- Declare cross-platform string utilities (`strupr`, `strlower`, `Q_stricmp`, etc.)
- Declare safe file I/O wrappers (`SafeOpenRead/Write`, `SafeRead/Write`, `LoadFile`, `SaveFile`)
- Declare path manipulation utilities (`ExpandPath`, `ExtractFilePath`, `DefaultExtension`, etc.)
- Provide endian-swap function declarations (`BigShort`, `LittleShort`, `BigLong`, etc.)
- Expose global game/tool directory state (`qdir`, `gamedir`, `writedir`)
- Declare CRC utility functions for data integrity checks
- Provide `qprintf`/`_printf` verbosity-gated output and fatal `Error` reporting

## External Dependencies
- **Standard C library:** `<stdio.h>`, `<string.h>`, `<stdlib.h>`, `<errno.h>`, `<ctype.h>`, `<time.h>`, `<stdarg.h>`
- **MSVC-specific:** `#pragma intrinsic(memset, memcpy)`, several `#pragma warning(disable)` suppressions
- All declared functions are **defined elsewhere** (in `common/cmdlib.c` or platform-specific translation units)

# common/imagelib.c
## File Purpose
A build-tool/offline image I/O library (used by q3map, bspc, q3radiant, etc.) providing load and save routines for four legacy 2-D image formats: LBM (IFF-ILBM/PBM), PCX, BMP, and TGA. It is not part of the runtime engine; it runs on the host machine during asset processing.

## Core Responsibilities
- Read and decode LBM (PBM packed variant) files including RLE decompression
- Write LBM (PBM) files with FORM/BMHD/CMAP/BODY IFF chunks
- Read and decode PCX (ZSoft RLE) files
- Write PCX files (minimal RLE encoding)
- Read BMP files (BitmapInfo 40-byte and BitmapCore 12-byte headers, 8-bit only)
- Read TGA files (types 2/3/10, 24/32-bit) from file or in-memory buffer; write uncompressed 32-bit TGA
- Provide unified dispatch functions (`Load256Image`, `Save256Image`, `Load32BitImage`) that select the format by file extension

## External Dependencies
- `cmdlib.h` — `LoadFile`, `SaveFile`, `Error`, `ExtractFileExtension`, `Q_stricmp`, `BigShort`, `BigLong`, `LittleShort`, `LittleLong`, `qboolean`, `byte` — all defined in `common/cmdlib.c`
- `imagelib.h` — declares all public functions in this file
- Standard C: `stdio.h`, `stdlib.h` (via cmdlib), `string.h` (`memset`, `memcpy`)

# common/imagelib.h
## File Purpose
Public header declaring image I/O utility functions used by offline tools (map compiler, BSP tools, editor). It provides a unified interface for loading and saving paletted (8-bit) and true-color (32-bit) images across multiple formats.

## Core Responsibilities
- Declare loaders and writers for LBM (Deluxe Paint) format
- Declare loaders and writers for PCX (ZSoft) format
- Declare loaders and writers for TGA (Targa) format
- Provide format-agnostic wrappers (`Load256Image`, `Save256Image`) that dispatch by file extension
- Provide a unified 32-bit RGBA loader (`Load32BitImage`) abstracting format details

## External Dependencies
- No includes declared in this header; consumers must include it after standard type headers (e.g., `cmdlib.h` for `byte`).
- All function bodies defined elsewhere (likely `common/imagelib.c`).

# common/l3dslib.c
## File Purpose
Offline tool-time library (not runtime game code) for loading triangle mesh geometry from Autodesk 3DS binary files. It parses the hierarchical chunk-based 3DS format and outputs a flat array of explicit `triangle_t` structs for use by model/map build tools.

## Core Responsibilities
- Open and validate a `.3ds` binary file header
- Recursively parse the 3DS chunk tree, descending into relevant parent chunks (`MAIN3DS`, `EDIT3DS`, `EDIT_OBJECT`, `OBJ_TRIMESH`)
- Parse vertex list chunks (`TRI_VERTEXL`) into a temporary float vertex pool
- Parse face/index list chunks (`TRI_FACEL1`) into a temporary index array
- Convert indexed triangles to explicit (expanded) `triangle_t` structs once both chunks are available
- Return the resulting triangle list and count to the caller via out-parameters

## External Dependencies
- `<stdio.h>` — `FILE`, `fread`, `fopen`, `fclose`, `fseek`, `feof`, `fprintf`
- `cmdlib.h` — `Error` (fatal error with exit), `MAXTRIANGLES` constant (defined elsewhere in tool lib)
- `mathlib.h` — included but no math functions are directly called here
- `trilib.h` / `l3dslib.h` — declares `triangle_t` type and `Load3DSTriangleList` prototype
- `triangle_t` — defined elsewhere (likely `trilib.h` or a polyset header); **not defined in this file**
- `MAXTRIANGLES` — defined elsewhere in the tool common library; **not defined in this file**

# common/l3dslib.h
## File Purpose
Public header for the 3DS (3D Studio) file loader library. Exposes a single function for importing triangle geometry from Autodesk 3DS format files, used by map/model build tools.

## Core Responsibilities
- Declare the public API for loading triangle mesh data from `.3ds` files
- Bridge between the 3DS binary format and the engine's internal `triangle_t` representation

## External Dependencies
- `triangle_t` — defined elsewhere (likely `common/trilib.h` or `common/polyset.h`)
- No standard library headers included directly in this header

# common/mathlib.c
## File Purpose
Implements the shared 3D math primitive library used by Quake III Arena's offline tools (q3map, bspc, q3radiant, etc.). Provides vector, plane, matrix, and spatial utility operations used across the common tool infrastructure.

## Core Responsibilities
- Vector arithmetic: add, subtract, scale, negate, copy, dot product, cross product
- Vector normalization and length computation
- Plane construction from points and plane type classification
- Bounding box management (clear and expand)
- 3x3 matrix multiplication and point rotation around an arbitrary axis
- Normal-to-compact-encoding conversion (lat/long byte encoding)
- Color vector normalization (max-component scale)

## External Dependencies
- `#include "cmdlib.h"` — provides `qboolean`, `byte`, standard C library headers (`stdio`, `stdlib`, `string`, `math`)
- `#include "mathlib.h"` — self-header; defines macros (`DotProduct`, `VectorSubtract`, etc.), type aliases, and `PLANE_*` constants
- `#pragma optimize("p", on)` — Windows-only: enables floating-point consistency optimization to avoid cross-platform precision divergence in tool computations
- All math functions (`sqrt`, `cos`, `sin`, `atan2`, `acos`, `fabs`, `floor`) come from `<math.h>` via `mathlib.h`

# common/mathlib.h
## File Purpose
Header-only math library defining 3D vector types, constants, and utility function declarations for use across Quake III Arena's tools (BSP compiler, map tools, bot utilities). It provides the foundational linear algebra primitives shared by tool-side code, distinct from the runtime `q_shared.h` math used in-game.

## Core Responsibilities
- Define scalar and vector types (`vec_t`, `vec2_t`, `vec3_t`, `vec4_t`) with optional double precision
- Declare BSP-relevant plane side constants and plane type classification
- Provide fast inline vector operations via macros (`DotProduct`, `VectorAdd`, etc.)
- Declare function prototypes for non-trivial math operations (normalization, cross product, bounds)
- Declare plane construction and normal encoding utilities
- Declare point rotation utility

## External Dependencies
- `<math.h>` — standard C math (used by implementation in `mathlib.c`)
- `qboolean`, `byte` — defined elsewhere (likely `cmdlib.h` or a shared `q_shared.h` equivalent for tools)
- `vec3_origin` — defined in `common/mathlib.c`

# common/md4.c
## File Purpose
Implements the RSA Data Security MD4 message-digest algorithm, adapted for use in the Quake III engine. It exposes a single engine-facing utility function (`Com_BlockChecksum`) that produces a 32-bit checksum over an arbitrary memory buffer using MD4 as the underlying hash.

## Core Responsibilities
- Initialize, update, and finalize MD4 hash contexts
- Process 64-byte message blocks through three rounds of bitwise transforms
- Encode/decode between little-endian byte arrays and 32-bit word arrays
- Produce a 128-bit digest, then XOR-fold it into a single 32-bit checksum for engine use

## External Dependencies
- `<string.h>` — for `memcpy` and `memset` (the private `MD4_memcpy`/`MD4_memset` wrappers are declared but **not defined or called** in this file; the implementation uses CRT directly)
- `Com_BlockChecksum` — defined here, declared/used elsewhere in `qcommon` (defined elsewhere: callers in `common/`, `qcommon/files.c`, etc.)

# common/mutex.c
## File Purpose
Provides a thin, platform-abstracted mutex API used by the Quake III build tools (q3map, bspc, etc.) to protect shared state during multi-threaded work distribution. Exactly one platform implementation is compiled via preprocessor guards, with a no-op fallback for single-threaded or unsupported builds.

## Core Responsibilities
- Allocate and initialize platform-native mutex objects (`MutexAlloc`)
- Acquire a mutex lock (`MutexLock`)
- Release a mutex lock (`MutexUnlock`)
- Short-circuit all locking when `numthreads == 1` (returns `NULL`, lock/unlock ignore `NULL`)
- Provide a do-nothing fallback when no recognized platform is detected

## External Dependencies
- `cmdlib.h` — provides `Error()` (fatal error termination), used on OSF1 mutex init failure
- `threads.h` — provides `numthreads` global
- `mutex.h` — declares `mutex_t` and the three public function prototypes
- `<windows.h>` (WIN32) — `CRITICAL_SECTION`, `EnterCriticalSection`, `LeaveCriticalSection`, `InitializeCriticalSection`
- `<pthread.h>` (OSF1) — `pthread_mutex_t`, `pthread_mutex_lock/unlock/init`, `pthread_mutexattr_*`
- `<task.h>`, `<abi_mutex.h>`, `<sys/types.h>`, `<sys/prctl.h>` (IRIX) — `abilock_t`, `spin_lock`, `release_lock`, `init_lock`

# common/mutex.h
## File Purpose
Declares a minimal, platform-agnostic mutex abstraction used by the Quake III toolchain (map compiler, BSPC, etc.) to synchronize multi-threaded operations. It provides opaque handle allocation and lock/unlock primitives over whatever threading backend the platform supplies.

## Core Responsibilities
- Define the opaque `mutex_t` handle type
- Declare allocation of a new mutex
- Declare lock and unlock operations on a mutex handle

## External Dependencies
- No includes in this header.
- `MutexLock`, `MutexUnlock`, `MutexAlloc` — defined elsewhere (expected in `common/mutex.c`), with the actual platform implementation (pthreads, Win32 `CRITICAL_SECTION`, etc.) hidden behind the `void *` abstraction.

# common/polylib.c
## File Purpose
Implements a convex polygon (winding) library used by the offline BSP compilation and map-processing tools. Provides allocation, clipping, geometric queries, and convex-hull merging operations on `winding_t` polygons.

## Core Responsibilities
- Allocate and free `winding_t` polygon objects with optional single-threaded diagnostics
- Clip windings against planes, producing front/back fragments (`ClipWindingEpsilon`, `ChopWindingInPlace`, `ChopWinding`)
- Generate a maximal base winding for an arbitrary plane (`BaseWindingForPlane`)
- Compute geometric properties: area, bounds, center, plane equation
- Validate winding geometry for convexity, planarity, and degeneracy (`CheckWinding`)
- Classify a winding relative to a plane (`WindingOnPlaneSide`)
- Incrementally grow a coplanar convex hull (`AddWindingToConvexHull`)

## External Dependencies
- `cmdlib.h` — `Error()`, `qboolean`, standard C includes
- `mathlib.h` — `vec_t`, `vec3_t`, all vector/cross/dot macros and functions, `SIDE_*` constants
- `polylib.h` — declares `winding_t`, `MAX_POINTS_ON_WINDING`, `ON_EPSILON`
- `qfiles.h` — `MAX_WORLD_COORD`, `MIN_WORLD_COORD`, `WORLD_SIZE`
- `numthreads` — **defined elsewhere** (threading layer, e.g. `common/threads.c`)
- `malloc`, `free`, `memset`, `memcpy`, `printf` — C standard library

# common/polylib.h
## File Purpose
Declares the `winding_t` polygon primitive and the full suite of convex-polygon (winding) utility functions used throughout Quake III's BSP compiler (`q3map`), collision system, and BSPC tool. Windings represent convex polygons defined by an ordered list of 3D vertices and are the fundamental geometric primitive for CSG, BSP splitting, and portal generation.

## Core Responsibilities
- Define the `winding_t` structure and its size limit constant
- Declare allocation and deallocation functions for windings
- Declare plane-clipping and chopping operations (the core BSP split primitive)
- Declare geometric query functions (area, bounds, plane, side classification)
- Declare convex hull merging support
- Define the `ON_EPSILON` tolerance used across all plane-side tests

## External Dependencies
- `vec3_t`, `vec_t` — defined in `mathlib.h` / `q_shared.h` (defined elsewhere).
- `MAX_POINTS_ON_WINDING` (64) constrains all winding allocations; callers must not exceed this.
- `ON_EPSILON` (0.1) — overridable at compile time via makefile `-D` flag.

# common/polyset.h
## File Purpose
Defines data structures and function declarations for managing collections of triangulated polygon sets used in model/geometry processing tools. It serves as a shared header for offline tools (map compiler, model exporters) rather than the runtime engine.

## Core Responsibilities
- Define compile-time limits for triangle and polyset counts
- Declare the `triangle_t` primitive (geometry + normals + UVs per tri)
- Declare the `polyset_t` container grouping named triangles with a material
- Expose the polyset utility API (load, collapse, split, snap, normal computation)

## External Dependencies
- `vec3_t` — defined elsewhere (expected in `mathlib.h` or `q_shared.h`)
- `POLYSET_MAXTRIANGLES`, `POLYSET_MAXPOLYSETS` — self-contained constants defined here
- Include guard: `__POLYSET_H__`
- No standard library headers included directly; assumes `vec3_t` is already in scope via the including translation unit

# common/qfiles.h
## File Purpose
Defines all on-disk binary file format structures for Quake III Arena, covering QVM bytecode, image formats (PCX, TGA), skeletal/rigid mesh models (MD3, MD4), and the BSP map format. This header is explicitly shared between the game engine and toolchain utilities and must remain identical in both.

## Core Responsibilities
- Define magic numbers, version constants, and size limits for all Q3 file formats
- Declare packed on-disk structs for QVM executable headers
- Declare on-disk structs for PCX and TGA image headers
- Declare the full MD3 rigid-body animated model format (frames, tags, surfaces, vertices)
- Declare the full MD4 skeletal/weighted model format (bones, LODs, weighted vertices)
- Declare the BSP map format (header, 17 named lumps, all lump entry structs)
- Provide world-space coordinate limits and lightmap dimension constants

## External Dependencies

- No `#include` directives are present in this file; it depends on `vec3_t` and `byte` being defined by an enclosing translation unit (typically via `q_shared.h`) before inclusion.
- `vec3_t`, `byte` — defined in `q_shared.h`, used but not declared here.

# common/scriplib.c
## File Purpose
Implements a stack-based script/text tokenizer for the Quake III tools (q3map, bspc, q3radiant). It reads text files or in-memory buffers, tokenizes them token-by-token with support for `$include` directives, nested file inclusion, and structured matrix parsing/writing.

## Core Responsibilities
- Load script files from disk or parse directly from memory buffers
- Maintain a stack of up to 8 nested script contexts (for `$include` support)
- Skip whitespace, line/block comments (`;`, `#`, `//`, `/* */`)
- Tokenize input into the global `token[]` buffer (quoted and unquoted tokens)
- Track line numbers across file boundaries for error reporting
- Parse 1D/2D/3D float matrices from parenthesis-delimited token streams
- Write 1D/2D/3D float matrices back to a FILE in parenthesis-delimited format

## External Dependencies
- **`cmdlib.h`** — `Error`, `LoadFile`, `ExpandPath`, `qboolean`, `qtrue`/`qfalse`
- **`scriplib.h`** — declares all exported symbols; also pulls in `mathlib.h` for `vec_t`
- Standard C: `stdio.h` (FILE, printf, fprintf), `stdlib.h` (free), `string.h` (strcmp, strcpy), `atof`

# common/scriplib.h
## File Purpose
Public interface header for the tool-suite script/token parser used by offline map-compilation tools (q3map, bspc, q3radiant). It exposes a simple line-oriented tokenizer and matrix I/O helpers built on top of a single global parse cursor.

## Core Responsibilities
- Declare global state for the active parse cursor (`scriptbuffer`, `script_p`, `scriptend_p`, `token`, etc.)
- Expose file-based and memory-based script loading (`LoadScriptFile`, `ParseFromMemory`)
- Provide token-stream control: fetch, un-fetch, and lookahead (`GetToken`, `UnGetToken`, `TokenAvailable`)
- Expose exact-match token assertion (`MatchToken`)
- Provide 1-D, 2-D, and 3-D float matrix parsing from the token stream
- Provide symmetric 1-D, 2-D, and 3-D float matrix writing to a `FILE *`

## External Dependencies
- `common/cmdlib.h` — `qboolean`, `LoadFile`, `Error`, file utilities
- `common/mathlib.h` — `vec_t`, `vec3_t` (float/double scalar type used by matrix helpers)
- `<stdio.h>` (via `cmdlib.h`) — `FILE *` used by Write* functions

# common/surfaceflags.h
## File Purpose
Defines all content and surface flag bitmask constants shared between the game engine, tools (q3map, bspc), and the botlib. It serves as a single authoritative source for brush content types and surface property flags used across the entire Quake III Arena toolchain.

## Core Responsibilities
- Define bitmask constants for brush **content types** (`CONTENTS_*`)
- Define bitmask constants for **surface properties** (`SURF_*`)
- Act as a shared header synchronized across `common/`, `code/game/`, and tool directories
- Annotate bot-specific content types for AAS/botlib consumption
- Mark BSP-compiler-specific flags (hints, skips, lightmap behavior)

## External Dependencies
None. No includes. No external symbols.

---

### Flag Group Summary

| Group | Range | Consumer |
|---|---|---|
| `CONTENTS_SOLID` / liquids / fog | bits 0–6 | Engine collision, game logic |
| `CONTENTS_AREAPORTAL` | `0x8000` | BSP vis system |
| `CONTENTS_PLAYERCLIP` / `MONSTERCLIP` | `0x10000–0x20000` | Collision |
| Bot contents (`TELEPORTER`–`DONOTENTER`) | `0x40000–0x200000` | AAS/botlib |
| `CONTENTS_ORIGIN` | `0x1000000` | BSP pre-processing only; stripped before compile |
| Game-only body/corpse/trigger/nodrop | `0x2000000–0x80000000` | Server game logic |
| `SURF_*` physics/audio flags | bits 0–`0x2000` | Game physics, audio |
| `SURF_*` BSP/compiler flags | `HINT`, `SKIP`, `NODRAW`, `NOLIGHTMAP`, `LIGHTFILTER`, `ALPHASHADOW` | q3map compiler only |
| `SURF_*` render flags | `NODLIGHT`, `POINTLIGHT`, `SKY` | Renderer |

# common/threads.c
## File Purpose
Provides a platform-abstracted threading layer for the Quake III build tools (q3map, bspc, etc.). It implements a work-queue dispatcher pattern where multiple threads pull integer work items from a shared counter, with compile-time backends for Win32, OSF1, IRIX, and a single-threaded fallback.

## Core Responsibilities
- Maintain a global work-item dispatch counter shared across all threads
- Provide `ThreadLock`/`ThreadUnlock` around the dispatch counter (platform-specific mutex/critical section)
- Report progress as a 0–9 percentage pacifier during long operations
- Spawn and join N worker threads via `RunThreadsOn`, dispatching a callback per item
- Provide `RunThreadsOnIndividual` as a higher-level wrapper that sets `workfunction` and delegates
- Auto-detect thread count from hardware (Win32/IRIX) or use a fixed default (OSF1)

## External Dependencies
- `cmdlib.h` — `qboolean`, `Error`, `_printf`, `qprintf`, `I_FloatTime`
- `threads.h` — declares `numthreads`, `GetThreadWork`, `RunThreadsOn`, `RunThreadsOnIndividual`, `ThreadLock`, `ThreadUnlock`, `ThreadSetDefault`
- **Win32:** `<windows.h>` — `CRITICAL_SECTION`, `CreateThread`, `WaitForSingleObject`
- **OSF1:** `<pthread.h>` — `pthread_mutex_t`, `pthread_create`, `pthread_join`
- **IRIX:** `<task.h>`, `<abi_mutex.h>`, `<sys/prctl.h>` — `sprocsp`, `abilock_t`, `spin_lock`/`release_lock`

# common/threads.h
## File Purpose
Declares the public interface for the thread management system used by Quake III's offline tools (map compiler, BSP tools). Provides a simple work-queue threading model with a global lock for non-thread-safe operations.

## Core Responsibilities
- Expose thread count configuration via `ThreadSetDefault`
- Distribute discrete work items across threads via a shared work counter
- Provide a mutual exclusion primitive (`ThreadLock`/`ThreadUnlock`) for critical sections
- Abstract platform-specific threading behind a uniform function-pointer dispatch API

## External Dependencies
- `qboolean` — defined in `q_shared.h` or equivalent shared header; not defined here
- Implementation symbols (`numthreads`, mutex state, work counter) — defined in `common/threads.c`

# common/trilib.c
## File Purpose
A tool-time (offline/build) library for loading 3D triangle geometry from Alias triangle binary files (.tri format). It parses the proprietary Alias object-separated triangle format and populates polyset arrays used by the Quake III map/model build tools.

## Core Responsibilities
- Parse the Alias binary triangle file format (magic number validation, big-endian byte swapping)
- Handle the hierarchical object/group structure encoded via `FLOAT_START`/`FLOAT_END` sentinel values
- Read per-vertex position, normal, and UV data from disk into `triangle_t` structures
- Allocate and populate `polyset_t` arrays for downstream consumers
- Enforce hard limits on triangle and polyset counts, calling `Error()` on overflow

## External Dependencies
- `<stdio.h>` — `FILE`, `fread`, `fopen`, `fclose`, `feof`
- `cmdlib.h` — `Error()`, `BigLong()`, `strlwr()`
- `mathlib.h` — `vec3_t` (via polyset.h triangle types)
- `polyset.h` — `triangle_t`, `polyset_t`, `POLYSET_MAXTRIANGLES`, `POLYSET_MAXPOLYSETS`
- `trilib.h` — declares `TRI_LoadPolysets` (defined here)
- `BigLong` — byte-swap utility defined in cmdlib, not in this file

# common/trilib.h
## File Purpose
Header file declaring the interface for loading triangle/polyset data from Alias triangle files. It exposes a single loading function used by tools (e.g., model compilers) that need to import geometry from the Alias `.tri` format.

## Core Responsibilities
- Declare the public API for the Alias triangle file loader
- Expose `TRI_LoadPolysets` as the sole entry point for consumers of `trilib.c`

## External Dependencies
- `polyset_t` — struct defined elsewhere (likely `common/polyset.h`)
- Implementation resides in `common/trilib.c`

