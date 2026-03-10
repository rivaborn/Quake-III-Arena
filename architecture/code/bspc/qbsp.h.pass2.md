# code/bspc/qbsp.h — Enhanced Analysis

## Architectural Role

`qbsp.h` is the integration header for **BSPC** (an offline BSP→AAS compiler tool, orthogonal to the runtime engine). It aggregates all build-time constants, global state, and inter-module APIs needed to execute a four-phase pipeline: **(1) map loading** (parse binary BSP from any supported format: Q1/Q2/Q3/HL/SiN), **(2) CSG + brush partitioning**, **(3) portal/visibility analysis**, and **(4) AAS navigation mesh generation**. Unlike `qcommon.h` (runtime integration), BSPC operates entirely offline and reuses large portions of `code/botlib/` (AAS format, clustering, reachability) via a parallel `code/bspc/aas_*` implementation tree.

## Key Cross-References

### Incoming (who depends on this file)
- **All BSPC `.c` modules**: Every file in `code/bspc/` includes `qbsp.h` as its single integration point
  - Map loaders: `map.c`, `map_q1.c`, `map_q2.c`, `map_q3.c`, `map_hl.c`, `map_sin.c`
  - Geometry pipeline: `csg.c`, `brushbsp.c`, `portals.c`, `tree.c`
  - AAS compilation: `aas_create.c`, `aas_store.c`, `aas_map.c`, `aas_gsubdiv.c`, `aas_areamerging.c`, `aas_facemerging.c`, `aas_edgemelting.c`, `aas_prunenodes.c`
  - Diagnostics: `glfile.c`, `gldraw.c`, `leakfile.c`, `textures.c`, `bspc.c` (main)

### Outgoing (what this file depends on)
- **Local BSPC utilities**: `l_cmd.h`, `l_math.h`, `l_poly.h`, `l_threads.h`, `l_mem.h`, `l_utils.h`, `l_log.h`, `l_qfiles.h` (all in `code/bspc/` or shared)
- **botlib interface**: `code/botlib/l_script.h` (lexer for entity string parsing)
- **Entity framework**: `l_bsp_ent.h` (defines `entity_t`)
- **Format definitions**: `q2files.h` (on-disk Quake 2 BSP structures)
- **Platform**: `<io.h>` (Win32), `<malloc.h>` (generic allocation)
- **Forward declarations**: `quakefile_s` (from `l_qfiles.h`), `tmp_face_s` (AAS build artifact)

## Design Patterns & Rationale

### 1. **Strategy Pattern for Format Polymorphism**
The header enumerates `MAPTYPE_QUAKE1/2/3`, `HALFLIFE`, `SIN` and exports separate loader pairs (`Q1_LoadMapFile` / `Q1_LoadMapFromBSP`, etc.) for each. This avoids a monolithic parser; each format is isolated. A single `loadedmaptype` global tracks which format was last loaded, gating downstream operations (e.g., lighting calculations differ in SiN).

**Rationale**: BSPC is a **universal compiler tool**; supporting multiple games' map formats justified the extra code complexity.

### 2. **Global State Arrays + Counters**
`mapplanes[MAX_MAPFILE_PLANES]`, `mapbrushes[MAX_MAPFILE_BRUSHES]`, `brushsides[MAX_MAPFILE_BRUSHSIDES]` are flat, pre-allocated global arrays with `nummapplanes`, `nummapbrushes`, `nummapbrushsides` counters. This is **pre-dynamic-array** era C: simple, cache-friendly, but rigid upper bounds.

**Rationale**: 1990s C; compile-time sizing prevented runtime-growable lists. Also, BSPC processes one map at a time, so globals are acceptable.

### 3. **Build Flags as Control Flow**
Dozens of `extern qboolean` flags (`noprune`, `nodetail`, `nomerge`, `nocsg`, `create_aas`, etc.) modify pipeline behavior at runtime without branching code paths (gates implemented at module level or as no-op function calls). This is **feature toggling via globals**, not dependency injection.

**Rationale**: Quick prototyping and debugging; allows selective disabling of compilation steps without rebuilding.

### 4. **Recursive Tree Construction**
`BrushBSP()` and its subroutines (`SplitBrush`, `AllocNode`) recursively partition space, mirroring the spatial partitioning algorithm. Tree nodes (`node_t`) contain pointers to left/right children and a per-node plane (`planenum`).

**Rationale**: **BSP tree structure mirrors algorithm**; recursion naturally maps to tree traversal and construction.

### 5. **Dual AAS Implementation**
Code duplication across `code/botlib/aas_*.c` (runtime) and `code/bspc/aas_*.c` (offline). Both load/validate AAS binary format identically; BSPC adds compilation phases (e.g., `AAS_Create`, `AAS_GravitationalSubdivision`, `AAS_MergeAreas`) that runtime botlib doesn't need.

**Rationale**: Offline compilation and runtime loading have different concerns; sharing headers (`be_aas_*.h`) but forking implementation avoids runtime bloat.

## Data Flow Through This File

```
1. LOAD PHASE (map loaders in map_*.c)
   Input: Binary BSP file (Quake format-specific)
   ↓
   Populate: mapplanes[], mapbrushes[], brushsides[], map_texinfo[]
   Output: nummapplanes, nummapbrushes, nummapbrushsides counters filled

2. CSG + BSP PHASE (csg.c → brushbsp.c)
   Input: mapbrushes[] array
   ↓
   ProcessWorldBrushes()
     → MakeBspBrushList() (convert map brushes to bspbrush_t fragments)
     → ChopBrushes() (CSG subtraction)
     → BrushBSP() (recursive partitioning)
   ↓
   Output: tree_t* with node_t/portal_t tree

3. PORTALIZATION PHASE (portals.c)
   Input: tree_t from step 2
   ↓
   MakeTreePortals() → SplitNodePortals() → allocate portal_t per node
   FloodEntities() (leak detection)
   FillOutside() (mark exterior leaf)
   FloodAreas() → MarkVisibleSides() (compute face visibility)
   ↓
   Output: Each portal has face[2], side links, visibility flags

4. AAS COMPILATION PHASE (aas_create.c, aas_gsubdiv.c, aas_facemerging.c, etc.)
   Input: tree_t + visibility data from step 3
   ↓
   AAS_Create()
     → AAS_CreateAreas() (convert tree nodes → AAS areas)
     → AAS_GravitationalSubdivision() (subdivide jump/fall gaps)
     → AAS_MergeAreas() / AAS_MergeAreaFaces() (optimize)
     → AAS_PruneNodes() (strip non-walkable geometry)
   ↓
   Output: aasworld_t (binary AAS data structure)

5. OUTPUT (aas_file.c)
   AAS_WriteAASFile() writes aasworld_t to disk (.aas binary file)
```

Each phase reads the output of the previous; global state persists across phases.

## Learning Notes

### Engine Design (Late 1990s Era)
- **No virtual filesystems in map tools**: Map loaders directly read `.bsp` files by path; no abstraction layer (contrast to runtime engine's virtual FS in `qcommon/files.c`).
- **Conditional compilation for game variants**: `#ifdef SIN`, `#ifdef ME` embed game-specific logic without polymorphism (e.g., `lightinfo` field in `side_t` only exists for SiN).
- **Global state over parameters**: Nearly 20 global `qboolean` flags and 10+ global arrays; no config structs or context objects passed around.
- **Recursive algorithms for spatial structures**: BSP tree construction and AAS area subdivision both use recursion; no iterative worklist approach.

### AAS (Area Awareness System) Integration
- AAS is **not** a first-class subsystem in the runtime engine; it's bot-specific navigation data built offline and loaded by `botlib`.
- BSPC is the **only** code path that *generates* AAS. The runtime botlib only *consumes* it.
- The `code/bspc/aas_*.c` files are **not accessible** to the runtime engine; they're linked into the BSPC tool only.
- This separation allows `code/botlib/` to remain lightweight (no compiler phases, no tree manipulation, no optimization passes).

### Geometric Concepts (Modern Parallels)
- **Brush + side + winding model** is similar to a half-edge or winged-edge mesh, but simpler: each brush is a convex polytope, each side is a plane, each winding is a 2D convex polygon in plane space.
- **Portal** is akin to a **Portal Graph** or **Dual Graph** of the BSP tree: connects neighboring leaves, enables visibility queries.
- **AAS area** is a **convex polygon in 3D** (like a navmesh cell), but stored as a planar face list; reachability links are **directed edges** in a **navigation graph**.
- **Cluster** (in `be_aas_cluster.c`) is a **portal graph SCC** (strongly connected component), used for PVS-like visibility optimization.

### Why This File Is Central
- **Single include point**: Every BSPC module includes only `qbsp.h`, not individual `.h` files. This centralizes the API surface and makes dependencies clear.
- **Type definitions are co-located with globals**: `plane_t`, `side_t`, `node_t`, `portal_t` are used by nearly all downstream modules, so centralizing them avoids forward-declaration chains.
- **Build flags are declared here**: All the compile-time toggles (`noprune`, `nocsg`, etc.) are declared in one place for coherent feature control.

## Potential Issues

1. **Fixed-size global arrays may overflow on large maps**: `MAX_MAPFILE_PLANES=256k`, `MAX_MAPFILE_BRUSHES=65k`, `MAX_MAPFILE_BRUSHSIDES=524k` are hardcoded. A pathologically large map could exceed these bounds silently (no dynamic reallocation).

2. **Loose coupling between phases via globals**: The CSG phase populates `mapplanes[]`, but there's no explicit contract or validation that downstream phases consume them correctly. If a phase modifies a plane in-place, other phases could see stale data.

3. **Format-specific fields scattered across structs**: `#ifdef SIN` / `#ifdef ME` conditionals add fields to `side_t`, `node_t`, `face_t`, `portal_t`. This makes the code fragile to new format support; adding Half-Life or Doom 3 support would require more `#ifdef` blocks.

4. **Forward declarations leak implementation details**: `tmp_face_s` is forward-declared in `portal_t` but defined elsewhere; this couples the BSP struct to AAS internals. Clean separation would hide this.

5. **Global stack for recursion**: `nodestack[NODESTACKSIZE]` and `nodestackptr` manage explicit stack depth for iterative tree traversal (to avoid stack overflow), but there's no bounds checking before push—a deep tree could overflow the nodestack.
