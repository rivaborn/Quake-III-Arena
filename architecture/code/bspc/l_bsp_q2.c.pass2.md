# code/bspc/l_bsp_q2.c — Enhanced Analysis

## Architectural Role

This file is a **Q2 BSP I/O layer** for the BSPC offline compiler tool, completely isolated from the runtime engine. It bridges disk format (Q2 `.bsp` files) to in-memory representation used by the offline AAS (Area Awareness System) compilation pipeline. The file has no runtime dependencies and no frame-per-frame involvement; it's invoked once at tool startup to load a map, and again at tool shutdown to serialize results. Its sibling `l_bsp_q3.c` handles the same role for Q3 format, demonstrating a **multi-format BSP abstraction** pattern.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/be_aas_bspc.c`** – AAS compilation front-end calls `Q2_LoadBSPFile` during `AAS_ProcessMap`; inherits all global lump arrays for reachability analysis
- **`code/bspc/aas_create.c`** – Geometry analysis functions (`AAS_Create`, `AAS_CreateAreas_r`) directly read `dplanes`, `dvertexes`, `dedges`, `dfaces`, `dbrushes` during area-to-reachability conversion
- **`code/bspc/bspc.c`** – Main tool driver calls `Q2_LoadBSPFile` and `Q2_WriteBSPFile` at tool entry/exit; tracks overall compilation flow
- **`code/bspc/aas_map.c`** – Map-specific processing reads all brush and face data; `AAS_SetTexinfo` depends on texture fixup having run

### Outgoing (what this file depends on)
- **`l_cmd.h`, `l_mem.h`, `l_log.h`** (from `code/bspc/`) – Low-level I/O, memory, logging utilities
- **`l_poly.h`** – Winding geometry library (`BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding`, `WindingArea`)
- **`l_script.h`** (from `code/botlib/`) – Entity string parsing via `LoadScriptMemory`/`ParseEntity` (shared with bot lib)
- **`q2files.h`** – Q2 BSP format type definitions and magic constants
- **`l_bsp_ent.h`** – Entity array and count shared with generic entity parser

## Design Patterns & Rationale

### 1. **Namespace via Macros (Multi-Format Abstraction)**
```c
#define q2_dmodel_t dmodel_t
#define q2_dplanes dplanes
// ... 50+ aliases mapping q2_* to shared type/global names
```
This technique allows identical format handling code across Q1/Q2/Q3 variants without duplicating function bodies. Each format module (`l_bsp_q3.c`, `l_bsp_q1.c`) defines its own aliases, then includes shared logic (indirectly through the AAS compiler). Clever but fragile—changes to shared structures ripple across all variants.

### 2. **Maximal Upfront Allocation (Simple, Memory-Inefficient)**
All lumps allocated at `MAX_MAP_*` capacity in `Q2_AllocMaxBSP`, then freed in paired `Q2_FreeMaxBSP`. No dynamic resizing. This is ideal for a non-interactive tool (simple memory bookkeeping, no fragmentation) but wastes RAM for small maps. Runtime engine code (collision, renderer) uses similar patterns during level load, then bounds-check at runtime.

### 3. **Explicit Endian Swapping (Cross-Platform Correctness)**
The `Q2_SwapBSPFile` function walks every field of every lump, byte-swapping in-place via `LittleLong`, `LittleShort`, `LittleFloat`. Called **before** interpreting data on load, **after** writing on save. This is essential for cross-platform tools; a single missed swap field silently corrupts geometry. The `todisk` parameter handles the asymmetry: loading must preserve `dvis->numclusters` before swapping it (to know loop bounds in decompression); saving swaps first then writes.

### 4. **Texture Reference Fixup (Ad-Hoc Repair)**
`Q2_FixTextureReferences` detects and repairs a common BSP corruption: brush sides with invalid texture references. It constructs a winding for each brush side, then finds the best-overlapping rendered face and copies its `texinfo` index. This is **tool-specific logic** that would never appear in runtime code—only offline tools can afford O(brushes × faces) per-map processing.

## Data Flow Through This File

**Load Flow:**
```
Q2_LoadBSPFile(filename)
  ├─ LoadFile(filename, ...) → file buffer + header
  ├─ Q2_CopyLump(LUMP_*) → copy each lump to global array
  ├─ Q2_SwapBSPFile(false) → byte-swap in-place
  ├─ Q2_ParseEntities() → entity string → entities[] array
  ├─ Q2_FixTextureReferences() → repair broken brush textures
  └─ FreeMemory(header)
```

**Geometry Query Flow (during AAS compilation):**
```
Q2_BrushSideWinding(brush, side)
  ├─ BaseWindingForPlane(plane) → huge 4D winding
  ├─ loop: ChopWindingInPlace(plane, ...) → clip against all brush sides
  └─ return resulting winding (or NULL if fully clipped)

Q2_FaceOnWinding(face, winding)
  ├─ CopyWinding(winding)
  ├─ loop: ChopWindingInPlace(edge_plane, ...) → clip to face boundary
  └─ return overlap area
```

**Save Flow:**
```
Q2_WriteBSPFile(filename)
  ├─ Q2_SwapBSPFile(true) → byte-swap in-place for disk
  ├─ SafeOpenWrite(filename)
  ├─ write header (placeholder)
  ├─ loop: Q2_AddLump(lump) → serialize each lump
  ├─ seek back and rewrite header with final offsets
  └─ fclose
```

**Key state mutation:** All global lump arrays modified during load; preserved through compilation; swapped in-place during save (making globals unusable afterward).

## Learning Notes

### Idiomatic to Offline Tools
- **Batch processing mindset**: Load entire map → process → save. No incremental updates, no streaming. Simpler reasoning; higher memory cost.
- **Geometric predicates are foundational**: `InsideWinding`, `InsideFace`, `Q2_FaceOnWinding` appear in many geometric contexts (visibility, collision, reachability). Learning these operations is essential for 3D game engine foundations.
- **PVS encoding (RLE)**: The `Q2_CompressVis`/`Q2_DecompressVis` functions show simple but effective run-length encoding of sparse visibility data—a pattern used in many game engines.
- **Format compatibility challenges**: The Q2 format is subtly different from Q3 (signed vs. unsigned edge indices, different lump ordering). Multi-format support requires careful testing.

### Modern Contrasts
- **Vertices per-lump vs. vertex buffers**: This file loads all vertices into a flat `dvertex_t[]` array. Modern engines use GPU vertex buffers with layout descriptors; here, all interpretation is done on CPU.
- **Brush representation**: Quake uses half-space brushes (planes + sides); modern engines often use triangle meshes or constructive solid geometry (CSG). The `Q2_BrushSideWinding` construction is specific to half-space geometry.
- **Entity parsing**: Hardcoded entity string parsing via `l_script.c`. Modern engines use JSON or data-driven formats with code generation.

### Connections to Engine Subsystems
- **AAS Compilation Pipeline**: This file's output (loaded BSP) feeds into `code/botlib/` via `code/bspc/be_aas_bspc.c`. The botlib never directly loads BSPs; BSPC pre-computes navigation data.
- **Collision Model (qcommon/cm_load.c)**: Runtime engine uses nearly identical lump loading logic, but for Q3 format only and with runtime memory constraints.
- **Renderer (renderer/tr_bsp.c)**: Both tools and renderer parse BSP faces and textures, but runtime renderer adds dynamic lighting, shadows, and draw-list sorting.

## Potential Issues

1. **Silent Texture Fixup Failures**: `Q2_FixTextureReferences` may fail to find a matching face for a brush side (no error raised). The side's `texinfo` remains unset or invalid, potentially causing downstream crashes in area creation. Should log warnings for unmatchable sides.

2. **Endian Swap Asymmetry Risk**: The `todisk` parameter in `Q2_SwapBSPFile` inverts the swap order for vis data decompression. If a new lump is added and its decompression depends on swapped data, the logic must be carefully re-examined. No assertions guard this invariant.

3. **Memory Fragmentation (Minor)**: Allocating 14 large lumps sequentially may fragment heap if the allocator is naive. For offline tools this is negligible, but affects startup time.

4. **No Validation of Loaded Data**: `Q2_LoadBSPFile` reads the header magic and version but does not validate lump sizes, count arrays, or cross-references. A malformed BSP (e.g., face with invalid plane index) will crash during AAS compilation, not at load time, making debugging harder.

---

**Summary:** This file exemplifies offline tool architecture—simple, direct I/O-bound processing with maximal upfront allocation. Its geometric predicates are foundational to game engine math. The multi-format design via macros is pragmatic but creates maintenance burden. The texture fixup pass is a reminder that real-world data (BSP files from various editors) is often corrupt and requires repair.
