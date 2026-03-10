# code/bspc/q2files.h

## File Purpose
Defines the on-disk binary file formats for Quake 2 assets, including PAK archives, PCX images, MD2 skeletal models, SP2 sprites, WAL textures, and the Q2 BSP map format. It is a read-only format specification header used by the BSPC tool to load and interpret legacy Quake 2 map data for AAS (Area Awareness System) generation.

## Core Responsibilities
- Define PAK archive header and file-entry structures for Q2 asset packages
- Declare the MD2 triangle-model binary layout (header, frames, verts, UVs, GL commands)
- Declare the SP2 sprite format structures
- Declare the WAL mip-texture format
- Define the Q2 BSP binary layout: lumps, planes, nodes, leaves, brushes, faces, edges, visibility
- Provide content flags (`CONTENTS_*`) and surface flags (`SURF_*`) for Q2 brush/surface classification
- Establish upper design bounds (`MAX_MAP_*`) for all BSP lump arrays

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `dpackfile_t` | struct | Single file entry within a PAK archive (name, offset, length) |
| `dpackheader_t` | struct | PAK archive header (magic, directory offset/length) |
| `pcx_t` | struct | PCX image file header + inline pixel data |
| `dmdl_t` | struct | MD2 model file header (counts and byte offsets for all sub-lumps) |
| `daliasframe_t` | struct | One animation frame of an MD2 model (scale, translate, compressed verts) |
| `dtrivertx_t` | struct | Single compressed vertex in an MD2 frame (3 byte coords + normal index) |
| `dstvert_t` / `dtriangle_t` | struct | MD2 UV coordinate and triangle index records |
| `dsprite_t` / `dsprframe_t` | struct | SP2 sprite file header and per-frame descriptor |
| `miptex_t` | struct | WAL texture header with 4 mip levels and animation chain |
| `dheader_t` | struct | Q2 BSP file header (magic, version, lump directory) |
| `lump_t` | struct | Generic BSP lump descriptor (file offset + byte length) |
| `dmodel_t` | struct | BSP submodel bounds, origin, headnode, and face range |
| `dplane_t` | struct | BSP plane (normal, distance, axial type) |
| `dnode_t` | struct | BSP internal node (plane, children, AABB, face range) |
| `texinfo_t` | struct | Surface texture projection vectors, flags, value, name, animation chain |
| `dface_t` | struct | BSP face (plane, edge list, texinfo, lightmap styles/offset) |
| `dleaf_t` | struct | BSP leaf (contents, cluster, area, AABB, leafface/brush lists) |
| `dbrush_t` / `dbrushside_t` | struct | Convex brush and its bounding sides |
| `dvis_t` | struct | Visibility lump header (cluster count, PVS/PHS bit-vector offsets) |
| `dareaportal_t` / `darea_t` | struct | Area portal connectivity for Q2 area-based PVS |
| `dedge_t` / `dvertex_t` | struct | BSP edge (two vertex indices) and raw vertex position |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure data-definition header with no functions.

## Control Flow Notes
This header is consumed at load time by BSPC's Q2 BSP reader (`l_bsp_q2.c`). It plays no role in any frame or update loop; its structures are mapped directly onto memory-loaded file bytes and then converted into BSPC's internal representation for AAS compilation.

## External Dependencies
- No includes within this file; it depends on basic C types (`short`, `int`, `byte`, `float`, `char`) provided by the including translation unit's environment (typically via `qfiles.h` or a platform header).
- `byte` is assumed to be `unsigned char`, defined elsewhere (e.g., `q_shared.h`).
- `MAX_SKINNAME` (64) is defined locally and reused by both MD2 and SP2 structures.
- Content and surface flag constants (`CONTENTS_*`, `SURF_*`) mirror definitions in `q_shared.h` for the game module; `CONTENTS_Q2TRANSLUCENT` is explicitly renamed to avoid collision with the Q3 `CONTENTS_TRANSLUCENT` symbol.
