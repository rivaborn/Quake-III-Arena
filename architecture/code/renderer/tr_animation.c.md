# code/renderer/tr_animation.c

## File Purpose
Implements skeletal animation (MD4 model format) for the Quake III renderer. It handles both the front-end surface submission and the back-end per-frame vertex skinning via weighted bone transforms.

## Core Responsibilities
- Register MD4 animated surfaces into the draw surface list (front-end)
- Interpolate bone matrices between two animation frames (lerp)
- Deform mesh vertices using weighted, multi-bone skeletal skinning
- Write skinned positions, normals, and texture coordinates into the tessellator (`tess`)
- Copy triangle index data into the tessellator index buffer

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `md4Header_t` | struct (extern, qfiles.h) | Top-level MD4 model header; holds frame/bone/LOD counts and offsets |
| `md4Surface_t` | struct (extern, qfiles.h) | Per-surface geometry data including vertex, triangle, and shader index |
| `md4LOD_t` | struct (extern, qfiles.h) | Level-of-detail block containing surface list for a given LOD |
| `md4Frame_t` | struct (extern, qfiles.h) | Per-frame bone array; variable-size, addressed by pointer arithmetic |
| `md4Bone_t` | struct (extern, qfiles.h) | 3×4 rotation/translation matrix for one bone |
| `md4Vertex_t` | struct (extern, qfiles.h) | Vertex with normal, texcoords, and a variable-length weight list |
| `md4Weight_t` | struct (extern, qfiles.h) | Bone index, weight scalar, and local-space offset for one influence |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `tess` | `shaderCommands_t` | global (extern) | Tessellator buffer; destination for skinned vertex/index data |
| `backEnd` | `backEndState_t` | global (extern) | Back-end state; supplies `currentEntity` with frame indices and backlerp |
| `tr` | `trGlobals_t` | global (extern) | Renderer globals; supplies `currentModel` for front-end LOD lookup |

## Key Functions / Methods

### R_AddAnimSurfaces
- **Signature:** `void R_AddAnimSurfaces( trRefEntity_t *ent )`
- **Purpose:** Front-end pass — iterates all surfaces in the first LOD of the current MD4 model and submits each to the draw surface list.
- **Inputs:** `ent` — the reference entity being rendered (used implicitly via `tr.currentModel`)
- **Outputs/Return:** void
- **Side effects:** Calls `R_AddDrawSurf` for each surface, appending entries to the scene's draw surface list.
- **Calls:** `R_GetShaderByHandle`, `R_AddDrawSurf`
- **Notes:** Only LOD 0 is ever submitted here; no LOD selection logic is present in this file.

### RB_SurfaceAnim
- **Signature:** `void RB_SurfaceAnim( md4Surface_t *surface )`
- **Purpose:** Back-end surface callback — performs frame interpolation of bones and full CPU skeletal skinning, writing the result into `tess`.
- **Inputs:** `surface` — pointer to the MD4 surface to skin; `backEnd.currentEntity` provides frame indices and `backlerp`.
- **Outputs/Return:** void
- **Side effects:** Writes to `tess.indexes`, `tess.xyz`, `tess.normal`, `tess.texCoords`; increments `tess.numIndexes` and `tess.numVertexes`.
- **Calls:** `RB_CheckOverflow`, `VectorClear`, `DotProduct`
- **Notes:**
  - Frame size is computed via zero-pointer trick: `(int)(&((md4Frame_t *)0)->bones[numBones])`.
  - When `backlerp == 0`, the current frame's bone array is used directly with no allocation; otherwise, a local stack array `bones[MD4_MAX_BONES]` holds the lerped result.
  - Vertex struct is variable-size (trailing `weights[]` array); pointer advance uses `&v->weights[v->numWeights]` rather than a fixed stride.
  - A commented-out `+12` offset exists referencing a TFC skeleton compatibility hack.
  - Skinning formula per axis: `sum(weight * (dot(boneRow, offset) + boneRow[3]))`.

## Control Flow Notes
- **Front-end (R_AddAnimSurfaces):** Called from the entity rendering path (alongside `R_AddMD3Surfaces`) when `tr.currentModel->type == MOD_MD4`. Runs once per entity per frame during scene graph traversal.
- **Back-end (RB_SurfaceAnim):** Registered in `rb_surfaceTable[SF_MD4]`. Invoked by the back-end draw loop when flushing tessellated geometry. Executes on each draw call after sorting.

## External Dependencies
- **Includes:** `tr_local.h` (brings in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `md4Header_t`, `md4Surface_t`, `md4LOD_t`, `md4Frame_t`, `md4Bone_t`, `md4Vertex_t`, `md4Weight_t` — defined in `qfiles.h`
  - `tess` (`shaderCommands_t`) — defined/owned by `tr_shade.c` / `tr_backend.c`
  - `backEnd` (`backEndState_t`) — defined in `tr_backend.c`
  - `tr` (`trGlobals_t`) — defined in `tr_init.c`
  - `R_AddDrawSurf`, `R_GetShaderByHandle`, `RB_CheckOverflow` — defined in other renderer modules
  - `VectorClear`, `DotProduct` — macros from `q_shared.h`
