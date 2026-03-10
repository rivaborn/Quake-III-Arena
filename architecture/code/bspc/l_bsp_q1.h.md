# code/bspc/l_bsp_q1.h

## File Purpose
Defines the on-disk data structures, lump layout constants, capacity limits, and extern declarations for Quake 1 BSP (version 29) files. It serves as the Q1 BSP format interface used by the BSPC tool to load, inspect, and convert Q1 maps into AAS data.

## Core Responsibilities
- Define Q1 BSP v29 format limits (map capacity constants)
- Declare all on-disk BSP lump structs (`q1_dnode_t`, `q1_dface_t`, `q1_dleaf_t`, etc.)
- Define lump index constants and the file header layout
- Expose global arrays holding the parsed BSP data to translation units
- Declare the BSP I/O and entity parsing API functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `q1_lump_t` | struct | File offset + length descriptor for a single BSP lump |
| `q1_dheader_t` | struct | BSP file header: version field + array of 15 lump descriptors |
| `q1_dmodel_t` | struct | Submodel (brush entity) record: bounds, origin, hull roots, face range |
| `q1_dmiptexlump_t` | struct | Texture lump header; offset table into embedded miptex data |
| `q1_miptex_t` | struct | Single miptex entry: name, dimensions, 4-level mip offsets |
| `q1_dvertex_t` | struct | Single 3D vertex (float[3]) |
| `q1_dplane_t` | struct | BSP splitting plane: normal, distance, axis type |
| `q1_dnode_t` | struct | Interior BSP node: plane index, two child refs, face range, AABB |
| `q1_dclipnode_t` | struct | Clip hull node (hulls 1–3): plane + two children (negative = contents) |
| `q1_texinfo_t` | struct | Texture projection vectors (s/t) + miptex index + flags |
| `q1_dedge_t` | struct | Edge: two vertex indices (unsigned short) |
| `q1_dface_t` | struct | Polygon face: plane, edge list, texinfo, lightmap styles + offset |
| `q1_dleaf_t` | struct | BSP leaf: content type, visibility offset, surface list, ambient levels |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `q1_nummodels` / `q1_dmodels` | `int` / `q1_dmodel_t *` | global | Loaded submodel array |
| `q1_visdatasize` / `q1_dvisdata` | `int` / `byte *` | global | Raw PVS visibility data |
| `q1_lightdatasize` / `q1_dlightdata` | `int` / `byte *` | global | Raw lightmap samples |
| `q1_texdatasize` / `q1_dtexdata` | `int` / `byte *` | global | Raw texture lump |
| `q1_entdatasize` / `q1_dentdata` | `int` / `char *` | global | Raw entity string |
| `q1_numleafs` / `q1_dleafs` | `int` / `q1_dleaf_t *` | global | Loaded leaf array |
| `q1_numplanes` / `q1_dplanes` | `int` / `q1_dplane_t *` | global | Loaded plane array |
| `q1_numvertexes` / `q1_dvertexes` | `int` / `q1_dvertex_t *` | global | Loaded vertex array |
| `q1_numnodes` / `q1_dnodes` | `int` / `q1_dnode_t *` | global | Loaded BSP node array |
| `q1_numtexinfo` / `q1_texinfo` | `int` / `q1_texinfo_t *` | global | Loaded texinfo array |
| `q1_numfaces` / `q1_dfaces` | `int` / `q1_dface_t *` | global | Loaded face array |
| `q1_numclipnodes` / `q1_dclipnodes` | `int` / `q1_dclipnode_t *` | global | Loaded clip node array |
| `q1_numedges` / `q1_dedges` | `int` / `q1_dedge_t *` | global | Loaded edge array |
| `q1_nummarksurfaces` / `q1_dmarksurfaces` | `int` / `unsigned short *` | global | Mark-surface index array |
| `q1_numsurfedges` / `q1_dsurfedges` | `int` / `int *` | global | Surface-edge index array |

## Key Functions / Methods

### Q1_AllocMaxBSP
- Signature: `void Q1_AllocMaxBSP(void)`
- Purpose: Allocate all global BSP data arrays to their maximum capacities
- Inputs: None
- Outputs/Return: void
- Side effects: Heap-allocates all `q1_d*` pointer arrays
- Calls: Not inferable from this file
- Notes: Must be called before `Q1_LoadBSPFile`

### Q1_FreeMaxBSP
- Signature: `void Q1_FreeMaxBSP(void)`
- Purpose: Free all globally allocated BSP data arrays
- Inputs: None
- Outputs/Return: void
- Side effects: Frees heap memory for all `q1_d*` arrays
- Calls: Not inferable from this file

### Q1_LoadBSPFile
- Signature: `void Q1_LoadBSPFile(char *filename, int offset, int length)`
- Purpose: Read a Q1 BSP file (or embedded BSP at a byte offset) into the global arrays
- Inputs: `filename` — path to BSP; `offset`, `length` — byte range within file (for PAK support)
- Outputs/Return: void
- Side effects: Populates all `q1_num*` counts and `q1_d*` pointer arrays
- Calls: Not inferable from this file

### Q1_WriteBSPFile
- Signature: `void Q1_WriteBSPFile(char *filename)`
- Purpose: Write the current in-memory BSP data back to disk in Q1 format
- Inputs: `filename` — output path
- Outputs/Return: void
- Side effects: File I/O write
- Calls: Not inferable from this file

### Q1_PrintBSPFileSizes / Q1_ParseEntities / Q1_UnparseEntities
- `Q1_PrintBSPFileSizes`: Prints counts/sizes of all lumps for diagnostics
- `Q1_ParseEntities`: Parses `q1_dentdata` string into structured entity list
- `Q1_UnparseEntities`: Serialises entity list back into `q1_dentdata` string

## Control Flow Notes
This header is an init-time interface. BSPC calls `Q1_AllocMaxBSP` → `Q1_LoadBSPFile` to ingest a Q1 map, then accesses the global arrays during AAS geometry compilation. It does not participate in any game frame loop.

## External Dependencies
- No standard library includes directly; relies on types (`byte`, `int`, `short`) from surrounding BSPC/Q3 shared headers
- All function bodies defined in `code/bspc/l_bsp_q1.c` (not shown)
- Guarded by `#ifndef QUAKE_GAME` — the extern declarations and function prototypes are excluded when building as an in-engine component
