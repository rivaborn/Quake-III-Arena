# Subsystem Overview

## Purpose
The `common/` subsystem is the shared offline-tool foundation for Quake III Arena's build and compile-time toolchain (q3map, bspc, q3radiant, q3asm, and related utilities). It provides portable utility infrastructure — math primitives, file I/O, geometry types, image I/O, threading, tokenization, and BSP data management — that all tool executables link against. It is not part of the runtime engine.

## Key Files

| File | Role |
|---|---|
| `cmdlib.c` / `cmdlib.h` | Core tool utility layer: file I/O, path manipulation, string utilities, endian conversion, CRC, error reporting, global `qdir`/`gamedir` state |
| `mathlib.c` / `mathlib.h` | 3D math primitives: vectors, planes, bounding boxes, matrix ops, normal encoding — tool-side equivalent of runtime `q_shared.h` math |
| `bspfile.c` / `bspfile.h` | BSP file I/O: owns all global BSP lump arrays, loads/writes BSP files with byte-swap, parses and serializes entity key/value data |
| `polylib.c` / `polylib.h` | Convex polygon (winding) library: allocation, plane-clipping, area/bounds queries, convex hull merging — core BSP split primitive |
| `scriplib.c` / `scriplib.h` | Stack-based script tokenizer: file and in-memory parsing, `$include` support, matrix I/O helpers |
| `cmdlib.c` / `cmdlib.h` | (see above) |
| `imagelib.c` / `imagelib.h` | Offline image I/O: load/save LBM, PCX, BMP, TGA; format-agnostic dispatch wrappers |
| `aselib.c` / `aselib.h` | ASE (Autodesk ASCII Scene Export) mesh loader: parses geometry, materials, animation frames into `polyset_t` arrays |
| `trilib.c` / `trilib.h` | Alias `.tri` binary mesh loader: populates `polyset_t` arrays from big-endian proprietary format |
| `l3dslib.c` / `l3dslib.h` | 3DS binary mesh loader: chunk-tree parser producing explicit `triangle_t` structs |
| `polyset.h` | Shared geometry container types: `triangle_t`, `polyset_t`, compile-time limits |
| `qfiles.h` | On-disk format structs for all Q3 file types: QVM, PCX, TGA, MD3, MD4, BSP — shared between engine and tools |
| `surfaceflags.h` | Authoritative bitmask definitions for `CONTENTS_*` and `SURF_*` flags; shared across engine, tools, and botlib |
| `threads.c` / `threads.h` | Platform-abstracted work-queue threading: dispatches integer work items across N worker threads (Win32, OSF1, IRIX, single-threaded fallback) |
| `mutex.c` / `mutex.h` | Thin platform-agnostic mutex abstraction used by threading and shared-state protection |
| `md4.c` | MD4 message-digest implementation; exposes `Com_BlockChecksum` for 32-bit data integrity checks |
| `mathlib.c` / `mathlib.h` | (see above) |

## Core Responsibilities

- Provide a portable C utility foundation (`cmdlib`) that all tool executables depend on for file I/O, path resolution, error handling, endian conversion, and string operations
- Own and manage all global BSP lump arrays; serialize and deserialize BSP files with correct endian handling; parse and expose entity key/value data
- Define the shared geometry primitive hierarchy: `vec3_t` scalars → `triangle_t` → `polyset_t` → `winding_t` — used uniformly across all mesh loaders and BSP tools
- Import 3D mesh geometry from multiple offline formats (ASE, Alias `.tri`, 3DS binary) into a common `polyset_t` representation
- Implement the convex polygon library (`winding_t`) that underpins BSP plane-splitting, portal generation, and CSG operations
- Tokenize tool script/config files with nested include support; parse and write structured matrix data for tool configuration
- Load and save paletted and true-color images (LBM, PCX, BMP, TGA) for texture and lightmap processing in offline tools
- Abstract multi-threaded work dispatch and mutual exclusion across Win32, OSF1, and IRIX platforms for parallelizing long compile operations
- Publish the single authoritative source of `CONTENTS_*` / `SURF_*` flag definitions consumed by engine, tools, and botlib

## Key Interfaces & Data Flow

**Exposed to other subsystems / tool executables:**
- `LoadFile` / `SaveFile` / `SafeOpenRead` / `SafeOpenWrite` and path utilities (via `cmdlib`) — consumed by every tool translation unit
- `LoadBSPFile` / `WriteBSPFile` / `ParseEntities` / `UnparseEntities` / key-value accessors (via `bspfile`) — consumed by q3map, bspc, q3radiant
- `winding_t` allocation, clipping, and query functions (via `polylib`) — consumed by q3map BSP splitter, portal generator, and bspc
- `polyset_t` / `triangle_t` geometry arrays populated by `aselib`, `trilib`, `l3dslib` — consumed by model compilers and map tools
- `GetToken` / `LoadScriptFile` / `ParseFromMemory` (via `scriplib`) — consumed by q3map shader parser, entity parser, and q3radiant
- `RunThreadsOn` / `RunThreadsOnIndividual` / `ThreadLock` / `ThreadUnlock` (via `threads`) — consumed by q3map lighting and vis passes
- `MutexAlloc` / `MutexLock` / `MutexUnlock` (via `mutex`) — consumed by threading layer and any shared-state tool code
- `Com_BlockChecksum` (via `md4`) — consumed by `qcommon/files.c` and other integrity-check sites
- `Load32BitImage` / `Load256Image` / `Save256Image` (via `imagelib`) — consumed by q3map lightmap and texture processing

**Consumed from other subsystems:**
- `qfiles.h` BSP lump structs (`dheader_t`, `dmodel_t`, `dsurface_t`, `drawVert_t`, etc.) — defined by `common/qfiles.h`, pulled into `bspfile` via `bspfile.h`
- `surfaceflags.h` `CONTENTS_*` / `SURF_*` constants — pulled into `bspfile.h` and any tool that needs brush classification
- `gamedir` / `qdir` global strings — set by tool `main()` entry points using `cmdlib` utilities; read by `aselib` for material path resolution and by `scriplib` for include resolution
- `numthreads` — set by tool argument parsing; read by `threads.c` and `mutex.c` to decide whether locking is needed

## Runtime Role

This subsystem has **no runtime engine role**. All files in `common/` are compiled exclusively into offline build tools (q3map, bspc, q3radiant, q3asm, etc.) that run on the host machine during asset and map compilation. There is no init/frame/shutdown lifecycle in the engine sense. Tool executables invoke `LoadBSPFile` / `WriteBSPFile`, `RunThreadsOn`, and geometry loaders as discrete sequential pipeline stages driven by tool `main()` functions, not by an engine frame loop.

The one exception is `md4.c`: `Com_BlockChecksum` is also called from `qcommon/files.c` in the runtime engine for pak file integrity checking, making `md4.c` the only file in `common/` with a dual tool-and-runtime presence.

## Notable Implementation Details

- **Global BSP lump arrays** in `bspfile.c` are flat C arrays with `MAX_MAP_*` compile-time limits (defined in `qfiles.h`); the entire BSP is loaded into these globals at once — there is no streaming or partial-load mechanism.
- **`scriplib`'s include stack** is fixed at depth 8 (`MAX_INCLUDES`); exceeding it produces a fatal `Error()`. The single global `token[]` buffer means the tokenizer is not reentrant.
- **`polylib`'s `winding_t`** is a fixed-size struct with a hard cap of 64 vertices (`MAX_POINTS_ON_WINDING`); windings exceeding this limit during clipping are a fatal error.
- **Threading is additive but optional**: `mutex.c` and `threads.c` both short-circuit all locking and thread spawning when `numthreads == 1`, making single-threaded tool builds fully functional without any platform threading support.
- **Endian handling** in `bspfile.c` is explicit and exhaustive: every lump struct field is individually byte-swapped on load and write, enabling the tools to run on big-endian build hosts (SGI IRIX, DEC Alpha) and produce correct little-endian BSP output.
- **`qfiles.h` is intentionally duplicated** between `common/` and `code/qcommon/`; `bspfile.h` selects the path via the `_TTIMOBUILD` preprocessor guard to allow both tool and engine builds to include it without circular path dependencies.
- **`mathlib.h`** uses `#pragma optimize("p", on)` under MSVC to force floating-point consistency across tool builds, preventing compiler-specific precision divergence from producing non-reproducible BSP geometry.
