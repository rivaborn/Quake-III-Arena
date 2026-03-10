# code/bspc/q3files.h

## File Purpose
Defines the binary file formats for Quake III Arena asset files used by the BSPC (BSP compiler) tool. It specifies on-disk data structures for MD3 triangle models and Q3 BSP map files, including all lump types, geometry limits, and layout constants.

## Core Responsibilities
- Define MD3 model format structures (frames, tags, surfaces, vertices, normals)
- Define Q3 BSP file format structures (header, lumps, nodes, leafs, brushes, surfaces)
- Enumerate all BSP lump indices and their count
- Provide capacity limits (`Q3_MAX_MAP_*`) for all BSP lump categories
- Provide capacity limits (`MD3_MAX_*`) for model geometry
- Define the draw surface type enum (`q3_mapSurfaceType_t`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `md3Frame_t` | struct | Per-frame bounding box, local origin, radius, and name for an MD3 model |
| `md3Tag_t` | struct | Named attachment point with origin and 3-axis orientation for MD3 |
| `md3Surface_t` | struct | MD3 sub-mesh header; holds counts and byte offsets for shaders, triangles, UVs, and XYZ/normals |
| `md3Shader_t` | struct | Shader name + in-game shader index for an MD3 surface |
| `md3Triangle_t` | struct | Three vertex indices forming one MD3 triangle |
| `md3St_t` | struct | Per-vertex UV texture coordinates for MD3 |
| `md3XyzNormal_t` | struct | Packed per-vertex position (short×3) and encoded normal (short) for MD3 |
| `md3Header_t` | struct | Top-level MD3 file header; counts and byte offsets for frames, tags, and surfaces |
| `q3_lump_t` | struct | File offset and byte length descriptor for one BSP lump |
| `q3_dheader_t` | struct | Q3 BSP file header containing ident, version, and array of 17 lumps |
| `q3_dmodel_t` | struct | BSP sub-model (inline model) with AABB and references to surfaces and brushes |
| `q3_dshader_t` | struct | BSP shader entry: name, surface flags, content flags |
| `q3_dplane_t` | struct | BSP plane: normal vector + distance |
| `q3_dnode_t` | struct | BSP interior node: plane index, two children, AABB |
| `q3_dleaf_t` | struct | BSP leaf: cluster, area, AABB, leaf-surface and leaf-brush index ranges |
| `q3_dbrushside_t` | struct | One side of a BSP brush: plane index and shader index |
| `q3_dbrush_t` | struct | BSP convex brush: first side, side count, shader index |
| `q3_dfog_t` | struct | Fog volume: shader name, brush index, visible clip side |
| `q3_drawVert_t` | struct | Renderable vertex: position, UV, lightmap UV, normal, RGBA color |
| `q3_mapSurfaceType_t` | enum | Surface geometry type: BAD, PLANAR, PATCH, TRIANGLE_SOUP, FLARE |
| `q3_dsurface_t` | struct | BSP draw surface: shader, fog, type, vert/index ranges, lightmap rect, patch dims |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header file containing only preprocessor constants, typedefs, and struct/enum definitions.

## Control Flow Notes
This file is a passive data-layout header. It is `#include`d by BSPC tool source files that read or write Q3 BSP and MD3 files from disk. It has no runtime logic of its own. The BSP lump index constants (`Q3_LUMP_*`) drive array indexing in BSP loader/writer code elsewhere. The `#ifndef __QFILES_H__` guard prevents double inclusion.

## External Dependencies
- No includes within this file; relies on `vec3_t` and `byte` being defined by an earlier include (typically `q_shared.h`) in any translation unit that includes this header.
- `vec3_t`, `byte`: defined elsewhere (q_shared.h / bg_public.h)

**Notes:**
- PCX and TGA struct definitions are commented out (block-comment syntax is broken — uses `* /` instead of `*/`), making them effectively dead documentation.
- MD3 normals use a compact spherical encoding packed into a single `short`.
- BSP plane pairs `(x&~1)` and `(x&~1)+1` are guaranteed opposites by convention (noted in comment).
- This file is nearly identical to `code/qcommon/qfiles.h` but with Q3-prefixed BSP names to avoid collisions in the BSPC tool, which also handles Quake 1/2/HL BSP formats.
