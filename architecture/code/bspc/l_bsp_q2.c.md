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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `dmodel_t` | typedef (via q2files.h) | BSP submodel descriptor |
| `dheader_t` | typedef | BSP file header with lump table |
| `dleaf_t` | typedef | BSP leaf node (holds cluster, area, face/brush refs) |
| `dface_t` | typedef | Rendered face (plane, edges, texinfo) |
| `dbrush_t` / `dbrushside_t` | typedef | Convex brush and its half-space sides |
| `dvis_t` | typedef | Visibility header (numclusters + bitofs table) |
| `winding_t` | struct (l_poly.h) | Convex polygon used for geometry operations |
| `texinfo_t` | typedef | Texture projection vectors and surface flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dmodels` / `nummodels` | `dmodel_t *` / `int` | global | Loaded BSP submodels |
| `dvisdata` / `dvis` / `visdatasize` | `byte *` / `dvis_t *` / `int` | global | Raw and typed vis data |
| `dlightdata` / `lightdatasize` | `byte *` / `int` | global | Lightmap lump |
| `dentdata` / `entdatasize` | `char *` / `int` | global | Entity string lump |
| `dleafs` / `numleafs` | `dleaf_t *` / `int` | global | Leaf array |
| `dplanes` / `numplanes` | `dplane_t *` / `int` | global | Plane array |
| `dvertexes` / `numvertexes` | `dvertex_t *` / `int` | global | Vertex array |
| `dnodes` / `numnodes` | `dnode_t *` / `int` | global | BSP node array |
| `texinfo[MAX_MAP_TEXINFO]` / `numtexinfo` | `texinfo_t[]` / `int` | global (static array) | Texture info (fixed array, not heap) |
| `dfaces` / `numfaces` | `dface_t *` / `int` | global | Face array |
| `dedges` / `numedges` | `dedge_t *` / `int` | global | Edge array |
| `dleaffaces` / `numleaffaces` | `unsigned short *` / `int` | global | Leaf-to-face index array |
| `dleafbrushes` / `numleafbrushes` | `unsigned short *` / `int` | global | Leaf-to-brush index array |
| `dsurfedges` / `numsurfedges` | `int *` / `int` | global | Signed surface-edge index array |
| `dbrushes` / `numbrushes` | `dbrush_t *` / `int` | global | Brush array |
| `dbrushsides` / `numbrushsides` | `dbrushside_t *` / `int` | global | Brush side array |
| `dareas` / `numareas` | `darea_t *` / `int` | global | Area array |
| `dareaportals` / `numareaportals` | `dareaportal_t *` / `int` | global | Area portal array |
| `brushsidetextured[MAX_MAP_BRUSHSIDES]` | `char[]` | file-static (global) | Per-brushside texture assignment flag |
| `bspallocated` / `allocatedbspmem` | `int` | global | Allocation tracking flags |
| `header` | `dheader_t *` | global | Points to file header during load/save |
| `wadfile` / `outheader` | `FILE *` / `dheader_t` | global | Output file handle and header buffer |

## Key Functions / Methods

### Q2_AllocMaxBSP
- **Signature:** `void Q2_AllocMaxBSP(void)`
- **Purpose:** Heap-allocates all BSP lump arrays at their maximum legal sizes, zeroed.
- **Inputs:** None
- **Outputs/Return:** None; populates all global lump pointers and count variables.
- **Side effects:** Calls `GetClearedMemory` for every lump; logs total allocated bytes via `Log_Print`.
- **Calls:** `GetClearedMemory`, `Log_Print`, `PrintMemorySize`
- **Notes:** `texinfo` is deliberately skipped (commented out) because it uses a static array.

### Q2_FreeMaxBSP
- **Signature:** `void Q2_FreeMaxBSP(void)`
- **Purpose:** Frees all heap-allocated BSP lump arrays and resets count variables to zero.
- **Side effects:** Calls `FreeMemory` on every pointer; logs freed byte count; resets `allocatedbspmem`.
- **Calls:** `FreeMemory`, `Log_Print`, `PrintMemorySize`

### Q2_LoadBSPFile
- **Signature:** `void Q2_LoadBSPFile(char *filename, int offset, int length)`
- **Purpose:** Loads a full Q2 BSP from disk: reads header, copies all lumps, byte-swaps, and fixes texture references.
- **Inputs:** `filename` — path to .bsp; `offset`/`length` — for embedded BSPs within a larger file.
- **Outputs/Return:** Populates all global lump arrays and count variables.
- **Side effects:** Allocates then frees `header` via `LoadFile`/`FreeMemory`; calls `Q2_SwapBSPFile`, `Q2_FixTextureReferences`.
- **Calls:** `LoadFile`, `Q2_CopyLump`, `FreeMemory`, `Q2_SwapBSPFile`, `Q2_FixTextureReferences`, `Error`
- **Notes:** Validates `IDBSPHEADER` magic and `BSPVERSION`.

### Q2_WriteBSPFile
- **Signature:** `void Q2_WriteBSPFile(char *filename)`
- **Purpose:** Serializes all in-memory BSP lumps to a .bsp file on disk.
- **Side effects:** Swaps data in-place (do not use globals after calling); opens/closes `wadfile`; writes header twice (once as placeholder, once with final offsets).
- **Calls:** `Q2_SwapBSPFile`, `SafeOpenWrite`, `SafeWrite`, `Q2_AddLump`, `fseek`, `fclose`

### Q2_SwapBSPFile
- **Signature:** `void Q2_SwapBSPFile(qboolean todisk)`
- **Purpose:** Byte-swaps every field of every BSP lump between little-endian disk format and host format, in-place.
- **Inputs:** `todisk` — if true, reads `dvis->numclusters` before swapping it (to know loop bound); if false, swaps first then loops.
- **Side effects:** Modifies all global lump arrays in-place.
- **Calls:** `LittleLong`, `LittleFloat`, `LittleShort`

### Q2_CompressVis / Q2_DecompressVis
- **Signature:** `int Q2_CompressVis(byte *vis, byte *dest)` / `void Q2_DecompressVis(byte *in, byte *decompressed)`
- **Purpose:** Run-length encode/decode PVS visibility rows (zero-byte RLE).
- **Inputs:** Raw/compressed vis row bytes; row length derived from `dvis->numclusters`.
- **Outputs:** Compressed length (compress) / decoded bytes written to `decompressed`.
- **Notes:** `DecompressVis` calls `Error` on a zero repeat count.

### Q2_FixTextureReferences
- **Signature:** `void Q2_FixTextureReferences(void)`
- **Purpose:** For each brush side lacking a valid texture, finds a rendered face coplanar with it and copies that face's `texinfo` index.
- **Side effects:** Writes `brushsidetextured[]`; modifies `dbrushsides[].texinfo`.
- **Calls:** `Q2_HintSkipBrush`, `Q2_BrushSideWinding`, `WindingIsTiny`, `WindingError`, `WindingArea`, `Q2_FaceOnWinding`, `FreeWinding`

### Q2_FaceOnWinding
- **Signature:** `float Q2_FaceOnWinding(q2_dface_t *face, winding_t *winding)`
- **Purpose:** Returns the area of intersection between a winding and a BSP face by clipping the winding to all face edge half-spaces.
- **Inputs:** Face and candidate winding.
- **Outputs:** Overlap area (0 if no overlap).
- **Calls:** `CopyWinding`, `ChopWindingInPlace`, `WindingArea`, `FreeWinding`

### Q2_BrushSideWinding
- **Signature:** `winding_t *Q2_BrushSideWinding(dbrush_t *brush, dbrushside_t *baseside)`
- **Purpose:** Constructs a winding for one brush side by clipping a huge base plane winding against all other sides of the brush.
- **Outputs:** Heap-allocated `winding_t *`, or NULL if fully clipped.
- **Calls:** `BaseWindingForPlane`, `ChopWindingInPlace`

### Q2_ParseEntities / Q2_UnparseEntities
- **Purpose:** Convert `dentdata` string to/from `entities[]` array using the script parser.
- **Calls:** `LoadScriptMemory`, `SetScriptFlags`, `ParseEntity`, `FreeScript` / string operations + `StripTrailing`

## Control Flow Notes
This file is a **BSP I/O layer** used at tool startup/shutdown, not per-frame. `Q2_AllocMaxBSP` → `Q2_LoadBSPFile` initializes all data; BSPC processes it; `Q2_WriteBSPFile` serializes results. There is no render or game loop involvement.

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
