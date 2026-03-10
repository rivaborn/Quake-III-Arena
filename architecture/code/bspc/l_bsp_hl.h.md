# code/bspc/l_bsp_hl.h

## File Purpose
Defines the on-disk BSP format structures and limits for Half-Life (GoldSrc engine) BSP version 30. It exposes extern declarations for all global BSP lump arrays and declares the utility API for loading, writing, and manipulating HL BSP files within the BSPC (BSP compiler/converter) tool.

## Core Responsibilities
- Define HL BSP v30 format constants (lump indices, version, map limits)
- Declare all BSP on-disk data structures (`hl_dmodel_t`, `hl_dnode_t`, `hl_dface_t`, etc.)
- Define leaf content type constants (`HL_CONTENTS_*`)
- Expose global extern arrays representing each loaded BSP lump
- Declare the HL BSP file I/O and utility API

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `hl_lump_t` | struct | File offset and byte length descriptor for a single BSP lump |
| `hl_dheader_t` | struct | BSP file header; holds version and array of 15 lump descriptors |
| `hl_dmodel_t` | struct | Submodel (brush entity) record; bounds, origin, hull root nodes, face range |
| `hl_dmiptexlump_t` | struct | Header for the texture lump; count + per-texture byte offsets |
| `hl_miptex_t` | struct | Single embedded texture with name, dimensions, and 4 mip-level offsets |
| `hl_dvertex_t` | struct | Single 3D vertex (float[3]) |
| `hl_dplane_t` | struct | BSP splitting plane; normal, distance, axis-alignment type |
| `hl_dnode_t` | struct | Interior BSP node; plane index, two children (nodes or leaves), face range, AABB |
| `hl_dclipnode_t` | struct | Clip hull node (simplified collision BSP); plane + two children |
| `hl_texinfo_t` | struct | Texture projection vectors (S/T axes + offsets) and miptex index |
| `hl_dedge_t` | struct | Edge defined by two vertex indices |
| `hl_dface_t` | struct | Polygon face; plane, edge list range, texinfo, lightmap style array, light data offset |
| `hl_dleaf_t` | struct | BSP leaf; content type, visibility offset, AABB, marksurface range, ambient sound levels |

## Global / File-Static State

All declared `extern`, guarded by `#ifndef QUAKE_GAME`.

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `hl_nummodels` / `hl_dmodels` | `int` / `hl_dmodel_t *` | global | Loaded submodel array |
| `hl_visdatasize` / `hl_dvisdata` | `int` / `byte *` | global | PVS visibility data blob |
| `hl_lightdatasize` / `hl_dlightdata` | `int` / `byte *` | global | Raw lightmap samples |
| `hl_texdatasize` / `hl_dtexdata` | `int` / `byte *` | global | Texture mip data blob |
| `hl_entdatasize` / `hl_dentdata` | `int` / `char *` | global | Entity string lump |
| `hl_numleafs` / `hl_dleafs` | `int` / `hl_dleaf_t *` | global | BSP leaf array |
| `hl_numplanes` / `hl_dplanes` | `int` / `hl_dplane_t *` | global | Splitting plane array |
| `hl_numvertexes` / `hl_dvertexes` | `int` / `hl_dvertex_t *` | global | Vertex array |
| `hl_numnodes` / `hl_dnodes` | `int` / `hl_dnode_t *` | global | BSP interior node array |
| `hl_numtexinfo` / `hl_texinfo` | `int` / `hl_texinfo_t *` | global | Texture info array |
| `hl_numfaces` / `hl_dfaces` | `int` / `hl_dface_t *` | global | Face array |
| `hl_numclipnodes` / `hl_dclipnodes` | `int` / `hl_dclipnode_t *` | global | Clip node array |
| `hl_numedges` / `hl_dedges` | `int` / `hl_dedge_t *` | global | Edge array |
| `hl_nummarksurfaces` / `hl_dmarksurfaces` | `int` / `unsigned short *` | global | Mark-surface index array |
| `hl_numsurfedges` / `hl_dsurfedges` | `int` / `int *` | global | Signed surface-edge index array |
| Per-lump `*_checksum` fields | `int` | global | CRC/checksum of each loaded lump for integrity verification |

## Key Functions / Methods

### HL_LoadBSPFile
- Signature: `void HL_LoadBSPFile(char *filename, int offset, int length)`
- Purpose: Reads a Half-Life BSP file from disk into the global lump arrays.
- Inputs: File path; byte offset and length within the file (for embedded BSPs).
- Outputs/Return: void; populates all global `hl_d*` arrays.
- Side effects: Allocates heap memory via `HL_AllocMaxBSP`; fills global state.
- Calls: Defined in `l_bsp_hl.c`.
- Notes: `offset`/`length` support reading a BSP embedded inside a larger file.

### HL_WriteBSPFile
- Signature: `void HL_WriteBSPFile(char *filename)`
- Purpose: Serializes the global lump arrays back to a BSP file on disk.
- Inputs: Destination file path.
- Outputs/Return: void; writes file.
- Side effects: File I/O.
- Calls: Defined in `l_bsp_hl.c`.

### HL_AllocMaxBSP / HL_FreeMaxBSP
- Signature: `void HL_AllocMaxBSP(void)` / `void HL_FreeMaxBSP(void)`
- Purpose: Allocate / free heap buffers for all global lump arrays at their maximum sizes.
- Side effects: Heap allocation/deallocation for all `hl_d*` pointers.

### HL_DecompressVis / HL_CompressVis
- Signature: `void HL_DecompressVis(byte *in, byte *decompressed)` / `int HL_CompressVis(byte *vis, byte *dest)`
- Purpose: Run-length decode/encode HL PVS (Potentially Visible Set) bitsets.
- Inputs: Compressed or raw vis byte stream.
- Outputs/Return: Decompressed bytes written in-place; `CompressVis` returns compressed byte count.

### HL_ParseEntities / HL_UnparseEntities
- Purpose: Convert the raw `hl_dentdata` entity string lump to/from structured key-value pairs used by BSPC tools.

### FastChecksum
- Signature: `int FastChecksum(void *buffer, int bytes)`
- Purpose: Compute a fast integrity checksum over an arbitrary memory buffer (used to populate `*_checksum` globals).

## Control Flow Notes
This is a **header-only interface file** for BSPC. At load time, `HL_AllocMaxBSP` is called first, then `HL_LoadBSPFile` fills all lump globals. The BSPC converter then reads these globals to build AAS (Area Awareness System) navigation data. `HL_WriteBSPFile` and `HL_FreeMaxBSP` are called during shutdown or after conversion. The entire API is conditionally compiled out when `QUAKE_GAME` is defined (i.e., inside the game VM).

## External Dependencies
- No explicit `#include` directives in this header; depends on surrounding build context to provide `byte`, `qboolean`, and basic C types.
- Implementation defined in `code/bspc/l_bsp_hl.c` (not shown here).
- `FastChecksum` is declared here but likely defined in a shared utility module.
- `hl_texinfo_t` reuses the same layout as Quake 2's texinfo (notable cross-format sharing).
