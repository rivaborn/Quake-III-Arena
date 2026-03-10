# code/bspc/l_bsp_sin.h — Enhanced Analysis

## Architectural Role

This file is a **format adapter** within BSPC's offline BSP-to-AAS compilation pipeline, specifically for Ritual Entertainment's Sin engine BSP format. It occupies the same structural role as `l_bsp_q3.h`, `l_bsp_q2.h`, etc.—a set of parallel global arrays and loaders that allow BSPC to process different engines' map formats interchangeably. The lump arrays declared here serve as a **staging area**: after `Sin_LoadBSPFile` populates them, the AAS generation machinery (in `code/botlib` and `code/bspc/aas_*`) reads these arrays to extract navigation topology, reachability, and entity metadata. Unlike the runtime engine's monolithic collision system (`qcommon/cm_load.c`), BSPC trades polymorphism for simplicity—each format has its own loader that directly populates format-specific globals.

## Key Cross-References

### Incoming (what calls these functions)

**From BSPC main compilation flow:**
- `code/bspc/bspc.c` and `code/bspc/be_aas_bspc.c` call `Sin_LoadBSPFile`, `Sin_AllocMaxBSP`, `Sin_FreeMaxBSP` during offline AAS compilation
- Dispatch is likely via format detection or command-line flag, selecting between `Sin_*`, `Q3_*`, `Q2_*` loaders

**From AAS generation pipeline:**
- `code/bspc/aas_*` files (reach, create, store, etc.) read all `sin_d*` globals when processing Sin maps
- `code/bspc/be_aas_bspc.c` bridges AAS routines and BSP loaders
- Example: `AAS_Create` (in `aas_create.c`) reads `sin_dnodes`, `sin_dfaces`, `sin_dleafs` to construct AAS areas

**From botlib (at runtime, if Sin maps are loaded):**
- `code/botlib/be_aas_file.c`, `be_aas_main.c` load pre-compiled `.aas` files (not `.bsp`), so *no direct dependency* on this header at runtime
- This header is BSPC-only; the bot-compiled AAS has already stripped BSP dependency

### Outgoing (what this file depends on)

- **`sinfiles.h`** — Defines all struct types (`sin_dmodel_t`, `sin_dvis_t`, `sin_dface_t`, etc.), lump indices (`SIN_LUMP_*`), magic constants, and map size limits
- **`l_bsp_sin.c`** — Implements all declared functions; performs binary file I/O, byte-order swapping, lump extraction, memory layout
- **Implicit platform/memory services** — Must call `malloc`/`free` (or `Hunk_Alloc`/similar) during `Sin_AllocMaxBSP` / `Sin_FreeMaxBSP`; file I/O via `FS_ReadFile` (if qcommon available) or direct `fopen` (if standalone BSPC)

## Design Patterns & Rationale

### Format Adapter Pattern
BSPC uses a **copy-paste adapter** architecture: `l_bsp_q3.h`, `l_bsp_q2.h`, `l_bsp_sin.h` are nearly identical skeletal structures, differing only in type names (`q3_dface_t` vs. `sin_dface_t`). This avoids:
- Virtual function overhead at offline compile-time
- Complex polymorphic initialization (BSPC is C, not C++)

**Tradeoff:** Maintenance burden (duplicate declarations) for simplicity and zero indirection.

### Global State for Staging
All BSP lumps are `extern` globals, not heap-wrapped objects. This is idiomatic for **offline tools**:
- BSP loading is monolithic per-map (not concurrent)
- No need for multiple simultaneous BSP contexts
- Simpler API: `Sin_LoadBSPFile(filename)` → all globals populated as side effects

### Two-Phase Allocation
`Sin_AllocMaxBSP()` pre-allocates all lumps to their maximum capacities *before* loading. This guarantees:
- No reallocation surprises during streaming
- Predictable memory footprint for build automation
- Clear shutdown semantics (`Sin_FreeMaxBSP`)

### Sin Format Extensions
The presence of `sin_lightinfo` (not in Q3) and `sin_dbrushsidetextured` indicates Sin BSP is a **Q3-compatible superset**:
- Same core lumps (nodes, faces, edges, etc.)
- Proprietary fields for Ritual's lighting model (sin_lightvalue_t)
- BSPC must handle these during AAS generation (likely ignored or used for hint brushes)

## Data Flow Through This File

1. **Initialization Phase**
   - `Sin_AllocMaxBSP()` reserves memory for all `sin_d*` arrays at `MAX_MAP_*` capacities
   - Globals are now ready to receive data

2. **Load Phase**
   - `Sin_LoadBSPFile(filename, offset, length)` opens the BSP file, reads header, swaps byte order if needed
   - Iterates through `SIN_LUMP_*` indices, extracts each lump into corresponding `sin_d*` array
   - Sets count globals (`sin_numfaces`, `sin_numleafs`, etc.)
   - Optional: `Sin_LoadBSPFileTexinfo()` for partial loads (supporting qdata tool reuse)

3. **Processing Phase**
   - AAS generation code (in `code/bspc/aas_create.c`, etc.) reads these globals
   - Extracts navigation topology: traces BSP tree, identifies walkable areas, computes reachability
   - Visibility data: `Sin_DecompressVis()` expands PVS/PHS clusters for line-of-sight hints

4. **Write/Unload Phase**
   - `Sin_WriteBSPFile()` serializes modified lumps back (post-compilation touch-ups)
   - `Sin_FreeMaxBSP()` deallocates all arrays at shutdown

## Learning Notes

### Idiomatic to BSPC / Offline Tool Design
- **No runtime polymorphism**: Format selection is compile-time or link-time, not dynamic
- **Monolithic globals**: Reflects pre-VM era when global state was acceptable for offline tools
- **Manual memory management**: No RAII; explicit alloc/free pairs expected from callers
- **Format-specific APIs**: Each game engine's BSP loader is self-contained; no generic BSP abstraction

### Quake-Engine Genealogy
- Sin is derived from Quake II's BSP format, extended with Q3-style improvements
- The PVS/vis compression (RLE-encoded) and cluster numbering are Quake1→Q2→Q3 heritage
- Entity parsing via raw string manipulation (not structured format) is a Quake-original pattern

### Modern Alternatives
- **Geometry abstraction layers** (e.g., USD in modern game editors) would decouple tool chains
- **Format plugins / reflection** could replace copy-paste adapters
- **Staged compilation** (BSP → intermediate → AAS) could hide format details from the AAS pipeline itself

### Key Insight: AAS Independence
After `Sin_LoadBSPFile()` populates these globals, the **AAS pipeline treats BSP as opaque geometry**: it queries areas, faces, nodes without caring that they came from Sin format. This suggests the `sin_d*` arrays could be replaced with any geometry source (e.g., `.obj` + collision markup); the architecture *could* be generalized but was never attempted (BSPC stayed laser-focused on game-engine BSP formats).

## Potential Issues

1. **No format version checking in header**: The magic (`SINGAME_BSPHEADER`, `SIN_BSPHEADER`) and version fields are checked in `l_bsp_sin.c`, but callers of this header may not validate. If a wrong format is passed, loading may silently read garbage or crash.

2. **Unbounded memory allocation**: `Sin_AllocMaxBSP()` allocates to `MAX_MAP_*` limits defined in `sinfiles.h`. If the limits are too conservative, large Sin maps fail; if too generous, BSPC wastes RAM. No dynamic resizing or fallback.

3. **Entity string re-parse overhead**: `Sin_ParseEntities()` likely tokenizes the raw `sin_dentdata` string every load. For large entity lists, this could be slow; however, for offline tools, this is acceptable.

4. **No thread safety**: All globals are shared; BSPC cannot parallelize across multiple Sin maps in a single process. Typical workaround: subprocess-per-map.

5. **Copy-paste maintenance burden**: Subtle divergences in `sinfiles.h` between Sin and Q3 formats could lead to silent mismatches. For instance, if Sin added a field to `sin_dface_t` but callers still use Q3-sized reads, data corruption ensues.
