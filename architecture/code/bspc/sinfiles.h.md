# code/bspc/sinfiles.h

## File Purpose
Defines the binary BSP file format structures and constants for the SIN engine (a Quake II-derived game). It is used by the BSPC tool to read and process SIN-format `.bsp` files for bot navigation area (AAS) generation.

## Core Responsibilities
- Define the SIN BSP version constant (`SINBSPVERSION 41`) and all map size upper bounds
- Declare lump index constants and the file header structure for SIN BSP files
- Provide geometry structures: planes, vertices, nodes, faces, leaves, edges, brushes
- Define SIN-specific surface flags (rendering, physics, material type) and content flags
- Declare the `sin_texinfo_t` structure with extended SIN-specific texture/surface properties
- Define visibility (`sin_dvis_t`) and area portal structures for PVS/PHS data
- Encode surface material types (wood, metal, stone, etc.) in the upper 4 bits of surface flags via `SURFACETYPE_FROM_FLAGS`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `sin_lump_t` | struct | Offset + length pair describing one lump within the BSP file |
| `sin_dheader_t` | struct | BSP file header: magic ident, version, and array of 20 lumps |
| `sin_dmodel_t` | struct | Submodel (inline brush model) bounds, origin, headnode, face range |
| `sin_dvertex_t` | struct | Single 3D vertex point |
| `sin_dplane_t` | struct | BSP splitting plane: normal, distance, axial type |
| `sin_dnode_t` | struct | Internal BSP node: plane, children, AABB, face range |
| `sin_texinfo_t` | struct | SIN-extended texture info: UV vectors, flags, name, physics props (friction, restitution, translucence), color, groupname |
| `sin_lightvalue_t` | struct | SIN light emission value with color, direct light angle/style |
| `sin_dedge_t` | struct | Edge defined by two vertex indices |
| `sin_dface_t` | struct | BSP face: plane, edge range, texinfo, lightmap styles/offset, SIN lightinfo |
| `sin_dleaf_t` | struct | BSP leaf: contents, cluster, area, AABB, leaf-face and leaf-brush ranges |
| `sin_dbrushside_t` | struct | One side of a brush: plane index, texinfo, SIN lightinfo |
| `sin_dbrush_t` | struct | Brush: first side, side count, contents flags |
| `sin_dvis_t` | struct | Visibility header: cluster count, PVS/PHS bit-vector offsets |
| `sin_dareaportal_t` | struct | Portal linking two areas |
| `sin_darea_t` | struct | Area: count and start of its portals |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header file containing only preprocessor definitions, typedefs, and struct declarations.

## Control Flow Notes
This file is included by SIN BSP loader/parser translation units within the BSPC tool (e.g., `l_bsp_sin.c`). It plays no active role in the frame loop; it is consumed at compile time to provide type layouts for binary file I/O during the offline map-to-AAS compilation process.

## External Dependencies
- `vec3_t` — defined in a shared math/types header (e.g., `mathlib.h` or `q_shared.h`); not defined here
- `byte` — platform typedef, defined elsewhere
- `MAXLIGHTMAPS` — redefined here to 16 (undefed first to override any prior definition)
- Conditional compilation entirely controlled by `#define SIN` (set at the top of this file itself)
