# code/bspc/l_bsp_q3.h — Enhanced Analysis

## Architectural Role

This header is the Q3 BSP data **interface module** for the BSPC offline compiler tool. It serves as the bridge between the raw BSP file parser (`l_bsp_q3.c`) and the AAS (Area Awareness System) compilation pipeline (`code/bspc/aas_*.c` and reused botlib routines). In the offline workflow, Q3 maps are deserialized into these globals, entities are parsed, and then the AAS subsystem reads this data to compute navigation structures written to `.aas` files for runtime bot pathfinding.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — Main BSPC tool entry point; orchestrates load → parse → compile → free sequence
- **`code/bspc/be_aas_bspc.c`** — AAS offline adapter; reads `q3_d*` globals (planes, nodes, leaves, faces, brushes, shaders) to build cluster/reachability/routing data
- **`code/bspc/aas_*.c` family** — All AAS compilation modules; reference these globals for geometric data, entity bounds, water/solid classification
- **`code/botlib/be_aas_bspq3.c`** — Runtime botlib BSP interface (also used during BSPC compilation); parses entity lumps via `AAS_ParseBSPEntities`

### Outgoing (what this file depends on)
- **`q3files.h`** — Provides all `q3_d*_t` struct definitions, lump indices, and max-size constants
- **`code/bspc/l_bsp_q3.c`** — Implements the three declared functions; allocates and populates all globals
- **Platform/VM abstraction** — Uses `struct quakefile_s` for filesystem/pak abstraction (defined elsewhere; likely in qcommon)

## Design Patterns & Rationale

### Global State as Shared Module Interface
This header exemplifies pre-C++ era multi-translation-unit C: **no namespacing, no encapsulation, pure extern globals**. The pattern `(count, pointer_array)` pairs (e.g., `q3_nummodels`/`q3_dmodels`) is idiomatic to Quake-era tools and runtime engines. It avoids the overhead of struct wrapping or vtable indirection in an offline, performance-less-critical tool.

### Lifecycle Management via Paired Init/Cleanup
The three functions form a clean contract:
- `Q3_LoadBSPFile` → allocates & populates
- `Q3_ParseEntities` → extracts secondary data
- `Q3_FreeMaxBSP` → deallocates everything

This mirrors botlib's `AAS_LoadMap` / `AAS_Shutdown` pattern and reflects the tool's single-map-at-a-time execution model.

### Dual-Role Header
This header is shared between two contexts:
1. **Offline (BSPC tool)**: populates globals, feeds AAS compiler
2. **Runtime (botlib)**: reads BSP file for entity queries (via `AAS_ParseBSPEntities`, etc.)

The `#include "q3files.h"` and commented-out `surfaceflags.h` suggest surface-flag constants come transitively or are not needed by this interface.

## Data Flow Through This File

**Input:**
- `struct quakefile_s *qf` → describes file path and optional pak context

**Processing (Q3_LoadBSPFile):**
1. Open/read each BSP lump from `qf`
2. Allocate heap arrays for geometry (planes, nodes, leaves, brushes, etc.) and rendering (verts, indexes, surfaces, shaders)
3. Deserialize binary lump data into struct arrays
4. Update all `q3_num*` counts
5. Byte-swap if needed (endianness)

**Intermediate (Q3_ParseEntities):**
- Parse `q3_dentdata` raw text into structured entity key-value pairs (implementation opaque from this header)

**Output:**
- All `q3_d*` pointers and counts become globally visible to downstream AAS compilation modules

**Cleanup (Q3_FreeMaxBSP):**
- `free()` all allocated arrays
- Zero all counts

## Learning Notes

### Quake Engine Archive Patterns
This header exposes the **Q3 BSP binary format** — a compile-time snapshot of a level's geometry, physics, and rendering data. The parallel structure with `code/bspc/l_bsp_q1.h`, `l_bsp_q2.h`, `l_bsp_hl.h`, and `l_bsp_sin.h` shows how BSPC is a **multi-format converter** — each game's BSP layout is abstracted by a per-format loader, and the rest of the pipeline (AAS compilation, area merging, reachability) is format-agnostic.

### Entity Lump as Text
The `q3_dentdata` / `q3_entdatasize` pair represents the **entity lump** — a freeform text buffer of `key "value"` pairs. This design (text in a binary file) is old-school and unusual but allows mappers to add arbitrary properties without changing the BSP binary format. The downstream `Q3_ParseEntities` call likely tokenizes this into a linked list or hash table.

### Shader Indirection
The `q3_dshaders` array and shader indices in surfaces/brushes show that **shaders are compile-time references**, not runtime strings. This enables fast shader lookups and LUT-based surface property queries.

### Memory Ownership
No Hunk allocator (used by the runtime engine); pure `malloc`. BSPC is a standalone tool with simpler lifetime requirements.

## Potential Issues

- **No thread-safety**: Global state + no locks means concurrent map loads would corrupt. Not a problem for a single-threaded offline tool, but makes BSPC unsuitable for server-side runtime compilation.
- **No validation bounds**: `Q3_LoadBSPFile` assumes the file is well-formed. Malformed lumps (truncated, overlapping) could cause out-of-bounds reads or crashes.
- **Memory leak risk**: If `Q3_ParseEntities` fails mid-parse, memory allocated by `Q3_LoadBSPFile` is orphaned unless error handling unwinds properly (not visible here).
- **Implicit endianness handling**: Byte-swapping logic is hidden in `l_bsp_q3.c`; no header-level hint of this complexity.
