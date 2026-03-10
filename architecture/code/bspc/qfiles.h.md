# code/bspc/qfiles.h

## File Purpose
Defines binary on-disk file formats for Quake 2-era assets used by the BSPC (BSP Compiler) tool. It covers PAK archives, PCX images, MD2 models, SP2 sprites, WAL textures, and the Q2 BSP map format. This is a legacy format header distinct from the Q3 BSP structures used at runtime.

## Core Responsibilities
- Define magic number constants (FourCC identifiers) for each file format
- Declare packed structs that map directly to binary file layouts
- Enumerate BSP lump indices and upper-bound limits for map data arrays
- Define surface flags (`SURF_*`) and content flags (`CONTENTS_*`) for brush/leaf classification
- Provide the complete Q2 BSP in-memory/on-disk structural hierarchy (header → lumps → geometry)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `dpackfile_t` | struct | Single file entry in a PAK archive (name, offset, length) |
| `dpackheader_t` | struct | PAK file header (magic, directory offset/length) |
| `pcx_t` | struct | PCX image header with palette and pixel data |
| `dmdl_t` | struct | MD2 model file header; stores counts and byte offsets to all data sections |
| `daliasframe_t` | struct | Single MD2 animation frame with scale, translate, and compressed vertex array |
| `dtrivertx_t` | struct | Per-vertex compressed position (3 bytes) and light normal index |
| `dsprite_t` / `dsprframe_t` | struct | SP2 sprite file header and per-frame descriptor |
| `miptex_t` | struct | WAL texture header with 4 mip-level offsets and animation chain |
| `dheader_t` | struct | Q2 BSP file header; holds magic, version, and 19 lump descriptors |
| `lump_t` | struct | File offset + length for a single BSP lump |
| `dmodel_t` | struct | BSP submodel (brush entity bounds, headnode, face range) |
| `dplane_t` | struct | BSP splitting plane (normal, dist, axial type) |
| `dnode_t` | struct | BSP internal node (plane, children, AABB, face range) |
| `dleaf_t` | struct | BSP leaf (contents, cluster/area, AABB, leafface/leafbrush ranges) |
| `dface_t` | struct | BSP face (plane, edges, texinfo, lightmap styles/offset) |
| `dbrush_t` / `dbrushside_t` | struct | Convex brush and its individual bevel sides |
| `texinfo_t` | struct | Texture projection vectors, flags, value, name, animation chain |
| `dedge_t` | struct | Edge defined by two vertex indices |
| `dvis_t` | struct | Visibility lump header; PVS/PHS bit-vector offsets per cluster |
| `darea_t` / `dareaportal_t` | struct | Area and its portals for audio/visibility area-graph |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header file containing only type definitions, `#define` constants, and struct declarations. No functions are defined here.

## Control Flow Notes
This file is consumed passively at compile time by BSPC source files that read or write Q2-format BSP data (e.g., `l_bsp_q2.c`). It plays no role in runtime frame/update loops. The structs serialize directly to/from disk via `fread`/`fwrite`-style I/O in the BSP compiler pipeline.

## External Dependencies
- No `#include` directives in this header; relies on the including translation unit to provide `byte`, `short`, `int` primitive typedefs (typically from `qfiles.h` or `q_shared.h` up the include chain).
- `MAX_SKINNAME` (64) is defined within this file and reused by both `dmdl_t` skin name storage and `dsprframe_t`.
- All `CONTENTS_*` and `SURF_*` flags are noted as needing to stay in sync with `q_shared.h` for the runtime engine.
