# common/qfiles.h

## File Purpose
Defines all on-disk binary file format structures for Quake III Arena, covering QVM bytecode, image formats (PCX, TGA), skeletal/rigid mesh models (MD3, MD4), and the BSP map format. This header is explicitly shared between the game engine and toolchain utilities and must remain identical in both.

## Core Responsibilities
- Define magic numbers, version constants, and size limits for all Q3 file formats
- Declare packed on-disk structs for QVM executable headers
- Declare on-disk structs for PCX and TGA image headers
- Declare the full MD3 rigid-body animated model format (frames, tags, surfaces, vertices)
- Declare the full MD4 skeletal/weighted model format (bones, LODs, weighted vertices)
- Declare the BSP map format (header, 17 named lumps, all lump entry structs)
- Provide world-space coordinate limits and lightmap dimension constants

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `vmHeader_t` | struct | QVM bytecode file header; describes code/data/BSS segment offsets and lengths |
| `pcx_t` | struct | PCX image file header for 8-bit paletted images |
| `TargaHeader` | struct | TGA image file header for 24/32-bit images |
| `md3Frame_t` | struct | Per-frame bounding box, origin, radius, and name for an MD3 model |
| `md3Tag_t` | struct | Named attachment point (origin + 3-axis matrix) per frame for model linking |
| `md3Surface_t` | struct | MD3 surface chunk header with offsets to shaders, triangles, UVs, and XYZ data |
| `md3Shader_t` | struct | Shader reference (name + in-game index) for an MD3 surface |
| `md3Triangle_t` | struct | Three vertex indices forming one MD3 triangle |
| `md3St_t` | struct | Per-vertex texture coordinate (UV) for MD3 |
| `md3XyzNormal_t` | struct | Compressed per-vertex position (short×3) and encoded normal (short) |
| `md3Header_t` | struct | Top-level MD3 file header; describes frame/tag/surface counts and offsets |
| `md4Weight_t` | struct | Bone index + weight contribution for one MD4 skinning influence |
| `md4Vertex_t` | struct | MD4 vertex with position, normal, UVs, and variable-length weight array |
| `md4Triangle_t` | struct | Three vertex indices forming one MD4 triangle |
| `md4Surface_t` | struct | MD4 surface chunk with bone-reference table and offsets |
| `md4Bone_t` | struct | 3×4 bone transform matrix for one bone in one MD4 frame |
| `md4Frame_t` | struct | MD4 frame with bounds, origin, radius, and inline bone array |
| `md4LOD_t` | struct | MD4 level-of-detail block referencing its surface list |
| `md4Header_t` | struct | Top-level MD4 file header with shared frame/bone data and per-LOD surfaces |
| `lump_t` | struct | File offset + byte length descriptor for one BSP lump |
| `dheader_t` | struct | BSP file header containing ident, version, and 17 `lump_t` entries |
| `dmodel_t` | struct | BSP sub-model (bounding box + surface/brush index ranges) |
| `dshader_t` | struct | BSP shader reference (path, surface flags, content flags) |
| `dplane_t` | struct | BSP splitting plane (normal + distance) |
| `dnode_t` | struct | BSP internal node (plane, two children, AABB) |
| `dleaf_t` | struct | BSP leaf node (cluster/area, AABB, leaf-surface and leaf-brush ranges) |
| `dbrushside_t` | struct | One side of a BSP brush (plane + shader) |
| `dbrush_t` | struct | BSP brush (side range + content shader) |
| `dfog_t` | struct | Fog volume referencing a brush and a visible clip side |
| `drawVert_t` | struct | Renderable vertex: position, UV, lightmap UV, normal, RGBA color |
| `mapSurfaceType_t` | enum | Surface geometry type: `BAD`, `PLANAR`, `PATCH`, `TRIANGLE_SOUP`, `FLARE` |
| `dsurface_t` | struct | BSP draw surface: shader, fog, type, vert/index ranges, lightmap info, patch dims |

## Global / File-Static State

None.

## Key Functions / Methods

None — this is a pure header file containing only type definitions, constants, and macros.

## Control Flow Notes

This file has no runtime presence. It is included at compile time by:
- The engine renderer (`tr_bsp.c`, `tr_model.c`) to read and interpret loaded file data.
- The collision/map loader (`cm_load.c`) for BSP lump parsing.
- The QVM loader (`vm.c`) for bytecode header validation.
- Tool-chain utilities (`q3map`, `bspc`, `q3radiant`) to write and read these formats at build time.
The comment "This file must be identical in the quake and utils directories" reflects that it straddles both runtime and offline tooling.

## External Dependencies

- No `#include` directives are present in this file; it depends on `vec3_t` and `byte` being defined by an enclosing translation unit (typically via `q_shared.h`) before inclusion.
- `vec3_t`, `byte` — defined in `q_shared.h`, used but not declared here.
