# code/qcommon/qfiles.h

## File Purpose
Defines the on-disk binary file formats for all major Quake III Arena asset types: QVM bytecode, image formats (PCX, TGA), skeletal/rigid 3D models (MD3, MD4), and BSP map data. This header must remain identical between the engine and tool utilities to ensure consistent parsing.

## Core Responsibilities
- Define magic numbers, version constants, and hard limits for each file format
- Declare packed structs that directly map to serialized on-disk layouts
- Provide BSP lump index constants and the BSP header/lump descriptor types
- Define MD3 (rigid keyframe) and MD4 (skeletal/weighted) model structures
- Define surface geometry types and per-vertex draw data for the BSP renderer
- Establish world-space coordinate bounds and lightmap dimensions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vmHeader_t` | struct | QVM bytecode file header; describes code/data/BSS segment offsets and sizes |
| `pcx_t` | struct | PCX image file header + inline pixel data (8-bit paletted) |
| `TargaHeader` | struct | TGA image file header (24/32-bit RGB/RGBA) |
| `md3Header_t` | struct | Top-level MD3 model file header with offsets to frames, tags, surfaces |
| `md3Frame_t` | struct | Per-frame bounding box, local origin, radius, and name |
| `md3Tag_t` | struct | Named attachment point with origin and 3-axis orientation per frame |
| `md3Surface_t` | struct | Per-surface chunk header with offsets to shaders, triangles, ST coords, XYZ normals |
| `md3Shader_t` | struct | Shader name + in-game index for an MD3 surface |
| `md3Triangle_t` | struct | Three vertex indices forming one triangle |
| `md3St_t` | struct | Per-vertex UV texture coordinates |
| `md3XyzNormal_t` | struct | Compressed per-vertex position (short×3) and packed normal (short) |
| `md4Header_t` | struct | Top-level MD4 skeletal model header; bone names, frames, LODs |
| `md4Frame_t` | struct | Per-frame bounds + array of bone matrices |
| `md4Bone_t` | struct | 3×4 bone transform matrix |
| `md4LOD_t` | struct | LOD chunk linking to its surfaces |
| `md4Surface_t` | struct | MD4 surface with weighted vertex and bone-reference offsets |
| `md4Vertex_t` | struct | Skeletal vertex: normal, UVs, weight count, variable-length weights |
| `md4Weight_t` | struct | Bone index, weight scalar, and offset vector for one influence |
| `lump_t` | struct | File offset and byte length of one BSP lump |
| `dheader_t` | struct | BSP file header: ident, version, array of 17 `lump_t` descriptors |
| `dmodel_t` | struct | BSP sub-model (brush model entity) with surface/brush ranges and bounds |
| `dshader_t` | struct | BSP shader reference: path, surface flags, content flags |
| `dplane_t` | struct | BSP split plane (normal + distance) |
| `dnode_t` | struct | BSP interior node: plane, children, AABB |
| `dleaf_t` | struct | BSP leaf: cluster/area index, surface and brush index ranges, AABB |
| `dbrushside_t` | struct | One side of a brush: plane index and shader index |
| `dbrush_t` | struct | Brush definition: first side, side count, content-determining shader |
| `dfog_t` | struct | Fog volume: shader path, brush reference, visible clip side |
| `drawVert_t` | struct | Per-vertex draw data: position, UVs, lightmap UVs, normal, RGBA color |
| `mapSurfaceType_t` | enum | Surface geometry kind: BAD, PLANAR, PATCH, TRIANGLE_SOUP, FLARE |
| `dsurface_t` | struct | BSP draw surface: shader, fog, type, vert/index ranges, lightmap info, patch dims |

## Global / File-Static State
None.

## Key Functions / Methods
None. This is a pure header file containing only type definitions, constants, and macros.

## Control Flow Notes
This file participates in no runtime control flow directly. It is included by:
- The renderer (`tr_bsp.c`, `tr_model.c`) to parse BSP and model data during map/model load.
- The collision system (`cm_load.c`) to read BSP lumps.
- The QVM loader (`vm.c`) to validate and map `vmHeader_t`.
- Tool utilities (`q3map`, `bspc`) to read/write BSP and model files at compile time.

All structs are designed for direct `memcpy`/pointer-cast deserialization from disk buffers, so field order and sizes are fixed by the format.

## External Dependencies
- No `#include` directives in this file; consumers must include `q_shared.h` first to supply `vec2_t`, `vec3_t`, and `byte` typedefs used within these structs.
- All symbols are self-contained definitions; nothing here references external functions.
