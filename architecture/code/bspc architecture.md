# Subsystem Overview

## Purpose
`code/bspc` is a standalone offline BSP Compiler (BSPC) tool that converts compiled BSP map files from multiple Quake-engine game formats (Q1, Q2, Q3, Half-Life, Sin) into AAS (Area Awareness System) binary navigation files. These `.aas` files encode the navigable geometry, reachability graph, and cluster data consumed by the bot AI at runtime.

## Key Files

| File | Role |
|---|---|
| `bspc.c` | Main entry point; owns global compilation flags, parses command-line arguments, dispatches to BSP→AAS / reachability / clustering / optimize pipelines |
| `qbsp.h` | Central shared header; defines all core BSP types (`plane_t`, `node_t`, `portal_t`, `tree_t`, `mapbrush_t`), global state declarations, and the full inter-module function API surface |
| `aas_create.c` / `aas_create.h` | Drives the full BSP-to-AAS conversion pipeline; owns temporary AAS data structures (`tmp_face_t`, `tmp_area_t`, `tmp_node_t`) |
| `aas_store.c` / `aas_store.h` | Finalizes the packed `aas_t` world from temporary structures; deduplicates vertices, edges, and planes via hash tables; owns the global `aasworld` |
| `aas_file.c` / `aas_file.h` | Serializes and deserializes the binary AAS file format; handles endian-swapping and XOR header obfuscation |
| `be_aas_bspc.c` / `be_aas_bspc.h` | BSPC-context adapter that stubs the `botlib_import_t` interface and drives AAS reachability and cluster computation via botlib code |
| `brushbsp.c` | BSP tree construction; split-plane selection heuristics, brush splitting, iterative tree building with thread-safe work queue |
| `csg.c` | Constructive Solid Geometry; carves intersecting brushes into non-overlapping convex sets before BSP construction |
| `map.c` | Central map data manager; plane deduplication, brush geometry construction, multi-format BSP load dispatch |
| `map_q3.c` / `map_q2.c` / `map_q1.c` / `map_hl.c` / `map_sin.c` | Per-format BSP-to-mapbrush converters for Q3, Q2, Q1, Half-Life, and Sin |
| `aas_areamerging.c` | Merges adjacent compatible AAS areas to reduce area count |
| `aas_facemerging.c` | Coalesces coplanar faces within and across areas |
| `aas_edgemelting.c` | Inserts shared boundary vertices between adjacent faces within an area |
| `aas_gsubdiv.c` | Gravitational and ladder-based geometric subdivision of areas |
| `aas_prunenodes.c` | Prunes redundant BSP tree nodes post-area-merge |
| `aas_map.c` | Expands map brushes by player bounding boxes and classifies brush contents for AAS |
| `portals.c` | Builds inter-leaf portals, flood-fills for leak detection, classifies BSP areas |
| `cfgq3.c` | DSL configuration file (not C code) defining Q3A physics and bounding-box parameters for AAS generation |
| `l_cmd.c` / `l_cmd.h` | File I/O, path utilities, byte-order swapping, argument parsing, CRC |
| `l_mem.c` / `l_mem.h` | Heap allocation with optional debug block tracking |
| `l_poly.c` / `l_poly.h` | Convex polygon (`winding_t`) full lifecycle: allocation, clipping, merging, validation |
| `l_threads.c` / `l_threads.h` | Cross-platform threading abstraction (Win32, pthreads/Linux, IRIX sproc) |
| `l_bsp_q3.c`, `l_bsp_q2.c`, `l_bsp_q1.c`, `l_bsp_hl.c`, `l_bsp_sin.c` | Per-format BSP binary file I/O with endian-swapping and entity lump parsing |
| `l_qfiles.c` / `l_qfiles.h` | Pak/PK3/ZIP asset discovery and loading abstraction |
| `gldraw.c` / `nodraw.c` | Windows-only GL debug visualization; `nodraw.c` is the headless no-op stub |
| `tetrahedron.c` | Converts AAS solid faces into a tetrahedral decomposition for spatial analysis |
| `aasfile.h` | Defines the AAS binary file format: header, 14 lumps, all `aas_*_t` structs, travel types, area/face flags |
| `qbsp.h` | (see above) |

## Core Responsibilities

- **Multi-format BSP ingestion**: Load and normalize BSP files from Quake 1, Quake 2, Quake 3, Half-Life, and Sin game formats into a unified internal `mapbrush_t` / `entity_t` representation.
- **CSG and BSP tree construction**: Perform Constructive Solid Geometry to produce non-overlapping convex brush sets, then build the binary space partition tree using plane-selection heuristics with optional multithreading.
- **BSP-to-AAS conversion pipeline**: Convert BSP leaf geometry into a temporary AAS world through sequential passes — face classification, area assignment, edge melting, face merging, area merging, gravitational and ladder subdivision, node pruning.
- **AAS geometry packing**: Deduplicate vertices, edges, and planes into contiguous indexed arrays; compute area bounds and centroids; serialize into the final `aas_t` world structure.
- **Reachability and cluster computation**: Delegate bot reachability analysis and area clustering to botlib routines via a stub adapter (`be_aas_bspc.c`) that replaces live engine services with offline equivalents.
- **AAS file I/O**: Read and write the binary AAS file format with endian-swapping and header obfuscation; support both current (`AASVERSION`) and legacy (`AASVERSION_OLD`) formats.
- **Physics configuration**: Parse a DSL configuration file (`cfgq3.c`) to populate gravity, friction, bounding-box, and movement-cost parameters that govern AAS area classification and reachability analysis.
- **Portal generation and leak detection**: Build inter-leaf portals, flood-fill from entity origins to detect map leaks, classify areas, and write `.lin` and `.prt` output files.

## Key Interfaces & Data Flow

**Exposed to other subsystems / consumers:**
- `.aas` binary files on disk, consumed at runtime by `code/botlib/` (specifically `be_aas_file.c`, `be_aas_route.c`, `be_aas_reach.c`, etc.) for bot navigation.
- `AAS_CalcReachAndClusters` (declared in `be_aas_bspc.h`) — entry point invoked by `bspc.c` to trigger botlib reachability and clustering computation from within the BSPC tool context.

**Consumed from other subsystems:**
- **`code/botlib/`**: `be_aas_cluster.c`, `be_aas_optimize.c`, `be_aas_reach.c`, `be_aas_def.h`, `aasfile.h`, `l_precomp.h`, `l_script.h`, `l_struct.h`, `l_libvar.h`, `l_memory.h` — the BSPC tool reuses botlib AAS pipeline code directly; `be_aas_bspc.c` provides stub implementations of `botlib_import_t` to satisfy botlib's engine abstraction layer.
- **`code/qcommon/cm_public.h`**: Collision model functions (`CM_BoxTrace`, `CM_PointContents`, `CM_InlineModel`, etc.) are wrapped by `be_aas_bspc.c` for use by botlib reachability code during offline compilation.
- **Input BSP files** from disk: the tool reads `.bsp` files produced by external map compilers (Q1/Q2/Q3/HL/Sin toolchains) and optional `.cfg` physics configuration files.

**Internal data flow summary:**
```
BSP file → format loader (l_bsp_q*.c) → map.c normalization
→ CSG (csg.c) → BSP tree (brushbsp.c) → portals (portals.c)
→ aas_create.c [face classification → area assignment]
→ aas_edgemelting → aas_facemerging → aas_areamerging
→ aas_gsubdiv → aas_prunenodes
→ aas_store.c [pack into aas_t]
→ be_aas_bspc.c [reachability + clustering via botlib]
→ aas_file.c → .aas file on disk
```

## Runtime Role

This subsystem has no runtime role in the game engine. It is an offline standalone tool invoked from the command line. Its lifecycle is:

- **Init**: `main()` in `bspc.c` parses command-line arguments, resolves file paths, loads the physics configuration via `LoadCfgFile`, and optionally opens a log file.
- **Execution**: Dispatches to one of six operations (BSP→MAP, BSP→AAS, reachability computation, cluster computation, AAS optimization, AAS info). The BSP→AAS path runs the full multi-pass conversion pipeline described above.
- **Shutdown**: Writes output files (`.aas`, `.map`, `.lin`, `.prt`), frees all allocated memory, and closes the log file. No persistent state survives between invocations.

## Notable Implementation Details

- **`cfgq3.c` is not C code**: Despite the `.c` extension, this file contains only a domain-specific script parsed by the botlib precompiler (`l_precomp.c` / `l_script.c`). It defines Q3A physics constants and bounding-box dimensions as data, not executable code.
- **Botlib reuse via stub adapter**: Rather than duplicating AAS reachability logic, `be_aas_bspc.c` satisfies botlib's `botlib_import_t` function table with stubs and wrappers around the offline collision model (`CM_*`), allowing botlib AAS code to run unchanged in the compiler context.
- **Two-phase area merging**: `aas_areamerging.c` runs grounded areas first, then all areas, iterating until no further merges occur, to maximize area consolidation while preserving ground/gap convexity invariants.
- **Multi-format BSP support with a unified plane table**: All format loaders normalize BSP geometry into a shared hash-deduplicated `mapplanes[]` table managed by `map.c`, enabling a single downstream pipeline regardless of source game.
- **GL debug visualization is Windows-only and build-selectable**: `gldraw.c` (local OpenGL window via `glaux`, or TCP socket to a remote GL server) is swapped at link time with `nodraw.c` (empty stubs) for headless builds. `nodraw.c` also defines the `drawflag`, `draw_mins`, and `draw_maxs` globals.
- **Threading is platform-dispatched at compile time**: `l_threads.c` contains four platform-specific backends (Win32, OSF1, Linux pthreads, IRIX sproc) selected by preprocessor guards, with a single-threaded fallback for all other targets.
- **Tetrahedral decomposition** (`tetrahedron.c`) is a spatial utility that converts AAS solid faces into a triangle mesh and then into tetrahedrons; it is invoked separately from the main AAS pipeline and not part of the standard BSP→AAS path.
- **Memory debug mode**: `l_mem.c` supports a `MEMDEBUG` build mode that maintains a doubly-linked list of tagged allocation headers for leak detection and block validation, with release mode falling back to a simple `malloc` wrapper with a byte counter.
