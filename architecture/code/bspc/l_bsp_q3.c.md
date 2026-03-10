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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `q3_dmodel_t` | struct (extern) | BSP submodel descriptor |
| `q3_dshader_t` | struct (extern) | Shader/texture reference |
| `q3_dleaf_t` | struct (extern) | BSP leaf node |
| `q3_dplane_t` | struct (extern) | Half-space plane (normal + dist) |
| `q3_dnode_t` | struct (extern) | BSP interior node |
| `q3_dbrush_t` | struct (extern) | Convex brush |
| `q3_dbrushside_t` | struct (extern) | One plane-side of a brush |
| `q3_drawVert_t` | struct (extern) | Draw vertex (xyz, normal, st, lightmap) |
| `q3_dsurface_t` | struct (extern) | Draw surface descriptor |
| `q3_dfog_t` | struct (extern) | Fog volume |
| `q3_dheader_t` | struct (extern) | BSP file header with lump table |
| `winding_t` | struct | Convex polygon used for brush side overlap tests |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `q3_nummodels` / `q3_dmodels` | `int` / `q3_dmodel_t *` | global | BSP submodels lump |
| `q3_numShaders` / `q3_dshaders` | `int` / `q3_dshader_t *` | global | Shader references lump |
| `q3_entdatasize` / `q3_dentdata` | `int` / `char *` | global | Entity string lump |
| `q3_numleafs` / `q3_dleafs` | `int` / `q3_dleaf_t *` | global | Leaf nodes lump |
| `q3_numplanes` / `q3_dplanes` | `int` / `q3_dplane_t *` | global | Planes lump |
| `q3_numnodes` / `q3_dnodes` | `int` / `q3_dnode_t *` | global | BSP nodes lump |
| `q3_numbrushes` / `q3_dbrushes` | `int` / `q3_dbrush_t *` | global | Brushes lump |
| `q3_numbrushsides` / `q3_dbrushsides` | `int` / `q3_dbrushside_t *` | global | Brush sides lump |
| `q3_numDrawVerts` / `q3_drawVerts` | `int` / `q3_drawVert_t *` | global | Draw vertices lump |
| `q3_numDrawSurfaces` / `q3_drawSurfaces` | `int` / `q3_dsurface_t *` | global | Draw surfaces lump |
| `q3_dbrushsidetextured` | `char[Q3_MAX_MAP_BRUSHSIDES]` | global | Per-side visibility flag array |
| `q3_surfaceplanes` | `q3_dplane_t *` | file-static (global linkage) | Precomputed planes for planar surfaces |
| `forcesidesvisible` | `qboolean` (extern) | global | Forces all brush sides to visible |

## Key Functions / Methods

### Q3_LoadBSPFile
- **Signature:** `void Q3_LoadBSPFile(struct quakefile_s *qf)`
- **Purpose:** Main BSP load entry point. Reads the file, validates ident/version, copies all lumps into global arrays, byte-swaps, then runs visible brush side detection.
- **Inputs:** `qf` — quake file descriptor (path + offset/length)
- **Outputs/Return:** void; populates all `q3_*` globals
- **Side effects:** Allocates all global lump arrays via `GetMemory`; frees the raw header buffer; calls `Q3_FindVisibleBrushSides`
- **Calls:** `LoadQuakeFile`, `Q3_SwapBlock`, `Q3_CopyLump`, `CountTriangles`, `FreeMemory`, `Q3_SwapBSPFile`, `Q3_FindVisibleBrushSides`
- **Notes:** Errors fatally on bad ident or version mismatch.

### Q3_WriteBSPFile
- **Signature:** `void Q3_WriteBSPFile(char *filename)`
- **Purpose:** Serializes all in-memory BSP lumps to a file, byte-swapping in place first, then rewriting the header at offset 0.
- **Inputs:** `filename` — output path
- **Outputs/Return:** void; writes file to disk
- **Side effects:** Byte-swaps global arrays via `Q3_SwapBSPFile`; file I/O
- **Calls:** `Q3_SwapBSPFile`, `SafeOpenWrite`, `SafeWrite`, `Q3_AddLump`, `fseek`, `fclose`

### Q3_FindVisibleBrushSides
- **Signature:** `void Q3_FindVisibleBrushSides(void)`
- **Purpose:** Marks each brush side as "textured/visible" if it overlaps a planar draw surface, filtering out internal and degenerate sides. Populates `q3_dbrushsidetextured`.
- **Inputs:** global brush and surface data
- **Outputs/Return:** void; sets `q3_dbrushsidetextured[]`
- **Side effects:** Allocates `q3_surfaceplanes` via `Q3_CreatePlanarSurfacePlanes`; winding allocations/frees; log output
- **Calls:** `Q3_CreatePlanarSurfacePlanes`, `Q3_BrushSideWinding`, `WindingIsTiny`, `WindingError`, `WindingArea`, `Q3_FaceOnWinding`, `FreeWinding`, `Log_Print`, `qprintf`

### Q3_BrushSideWinding
- **Signature:** `winding_t *Q3_BrushSideWinding(q3_dbrush_t *brush, q3_dbrushside_t *baseside)`
- **Purpose:** Creates a `winding_t` for one brush side by starting with a half-space polygon and clipping it against all other brush planes.
- **Inputs:** `brush` — the parent brush; `baseside` — the side to generate geometry for
- **Outputs/Return:** Allocated `winding_t *`, or NULL if clipped away
- **Calls:** `BaseWindingForPlane`, `ChopWindingInPlace`

### Q3_FaceOnWinding
- **Signature:** `float Q3_FaceOnWinding(q3_dsurface_t *surface, winding_t *winding)`
- **Purpose:** Returns the overlapping area between a draw surface polygon and a brush side winding, by clipping the winding to all edge planes of the surface.
- **Inputs:** `surface` — draw surface; `winding` — brush side winding
- **Outputs/Return:** float area of overlap (0 if none)
- **Calls:** `CopyWinding`, `Q3_SurfacePlane`, `ChopWindingInPlace`, `WindingArea`, `FreeWinding`

### Q3_FreeMaxBSP
- **Signature:** `void Q3_FreeMaxBSP(void)`
- **Purpose:** Frees all dynamically allocated global BSP lump arrays and resets counts to zero.
- **Calls:** `FreeMemory` (for each lump pointer)

### Q3_ParseEntities / Q3_UnparseEntities
- **Purpose:** Convert the entity lump string to/from the internal `entities[]` array using a script tokenizer.
- **Notes:** `Q3_UnparseEntities` writes back to `q3_dentdata` in place; errors if entity text exceeds `Q3_MAX_MAP_ENTSTRING`.

## Control Flow Notes
This file is used during BSPC's offline BSP-to-AAS conversion pipeline, not at game runtime. `Q3_LoadBSPFile` is the entry point called once during init-equivalent processing. After loading, `Q3_FindVisibleBrushSides` runs as a preprocessing pass. `Q3_WriteBSPFile` is an optional output step. There is no per-frame or update loop involvement.

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
