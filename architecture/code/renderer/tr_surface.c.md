# code/renderer/tr_surface.c

## File Purpose
Implements the renderer back-end surface tessellation dispatch layer for Quake III Arena. It converts every recognized surface type (BSP faces, grid meshes, triangle soups, MD3 meshes, sprites, beams, rails, lightning) into vertices and indices written into the global `tess` (shaderCommands_t) buffer for subsequent shader execution.

## Core Responsibilities
- Guard the tess buffer against overflow and flush/restart it via `RB_CheckOverflow`
- Emit billboard quads (sprites, flares) into the tess buffer
- Tessellate static BSP geometry: planar faces (`srfSurfaceFace_t`), grid/patch meshes (`srfGridMesh_t`), triangle soups (`srfTriangles_t`)
- Lerp and decode MD3 compressed vertex/normal data into the tess buffer (`LerpMeshVertexes`, `RB_SurfaceMesh`)
- Generate procedural geometry for special entity types: beams, rail core/rings, lightning bolts
- Dispatch to the correct tessellation function through the `rb_surfaceTable` function pointer array

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `shaderCommands_t` (tess) | struct (extern) | Global tessellation accumulator: xyz, normals, texcoords, colors, indexes |
| `srfSurfaceFace_t` | struct | BSP planar face with inline point/index storage |
| `srfGridMesh_t` | struct | Bézier/grid patch mesh with LOD error tables |
| `srfTriangles_t` | struct | Arbitrary triangle soup (misc_model geometry) |
| `srfPoly_t` | struct | Client-submitted convex polygon |
| `md3Surface_t` | struct | MD3 model surface with compressed short xyz/normals |
| `srfDisplayList_t` | struct | Pre-compiled OpenGL display list surface |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `rb_surfaceTable` | `void(*[])(void*)` | global | Jump table indexed by `surfaceType_t`; maps surface kind to its tessellation function |
| `tess` | `shaderCommands_t` (extern) | global | Shared tessellation buffer written by all surface functions |
| `backEnd` | `backEndState_t` (extern) | global | Current entity, view parameters, SMP frame index |
| `tr` | `trGlobals_t` (extern) | global | Renderer globals: sin table, white image, etc. |

## Key Functions / Methods

### RB_CheckOverflow
- **Signature:** `void RB_CheckOverflow(int verts, int indexes)`
- **Purpose:** Ensures space exists in `tess` for the requested verts/indexes; flushes and restarts if not.
- **Inputs:** Count of vertices and indices about to be added.
- **Outputs/Return:** void
- **Side effects:** May call `RB_EndSurface()` then `RB_BeginSurface()`, resetting `tess`.
- **Calls:** `RB_EndSurface`, `RB_BeginSurface`, `ri.Error`
- **Notes:** Hard-errors if a single batch exceeds `SHADER_MAX_VERTEXES/INDEXES`.

### RB_AddQuadStampExt
- **Signature:** `void RB_AddQuadStampExt(vec3_t origin, vec3_t left, vec3_t up, byte *color, float s1, float t1, float s2, float t2)`
- **Purpose:** Appends a view-facing quad (4 verts, 6 indices) with custom UV rect into `tess`.
- **Inputs:** World-space origin, half-extent vectors, RGBA color, UV corners.
- **Outputs/Return:** void
- **Side effects:** Increments `tess.numVertexes` (+4) and `tess.numIndexes` (+6).
- **Calls:** `RB_CHECKOVERFLOW`, `VectorSubtract`
- **Notes:** Normal is computed as the negated view axis[0] (camera-facing). `RB_AddQuadStamp` wraps this with full [0,1] UVs.

### RB_SurfaceSprite
- **Signature:** `static void RB_SurfaceSprite(void)`
- **Purpose:** Generates a billboard quad for `RT_SPRITE` entities, optionally rotated around the view axis.
- **Inputs:** `backEnd.currentEntity` (radius, rotation, origin, shaderRGBA).
- **Side effects:** Writes into `tess` via `RB_AddQuadStamp`.
- **Calls:** `VectorScale`, `VectorMA`, `RB_AddQuadStamp`

### RB_SurfaceTriangles
- **Signature:** `void RB_SurfaceTriangles(srfTriangles_t *srf)`
- **Purpose:** Copies a triangle soup's verts (xyz, normal, ST, lightmap ST, color) and indices into `tess`.
- **Side effects:** Accumulates `tess.dlightBits`; increments vertex/index counts.
- **Calls:** `RB_CHECKOVERFLOW`
- **Notes:** Normal copy is conditional on `tess.shader->needsNormal`.

### LerpMeshVertexes
- **Signature:** `static void LerpMeshVertexes(md3Surface_t *surf, float backlerp)`
- **Purpose:** Decompresses MD3 short-quantized xyz and latitude/longitude-encoded normals; lerps between `frame` and `oldframe` when `backlerp != 0`.
- **Inputs:** MD3 surface pointer, lerp fraction (0 = new frame only).
- **Side effects:** Writes to `tess.xyz` and `tess.normal` at current vertex offset.
- **Calls:** `VectorNormalizeFast`, `VectorArrayNormalize`, accesses `tr.sinTable`
- **Notes:** Contains Altivec (PPC) and generic C paths. Normals are not re-normalized after lerp in the scalar path (commented-out `VectorNormalize`); `VectorArrayNormalize` is called instead.

### RB_SurfaceGrid
- **Signature:** `void RB_SurfaceGrid(srfGridMesh_t *cv)`
- **Purpose:** Tessellates a curved patch mesh with view-distance LOD, potentially flushing and restarting `tess` mid-surface for very large grids.
- **Inputs:** Grid mesh with precomputed LOD error tables.
- **Side effects:** Multiple potential `RB_EndSurface`/`RB_BeginSurface` cycles; writes verts and quad-strip indices.
- **Calls:** `LodErrorForVolume`, `RB_EndSurface`, `RB_BeginSurface`

### RB_SurfaceEntity
- **Signature:** `void RB_SurfaceEntity(surfaceType_t *surfType)`
- **Purpose:** Dispatches procedural entity surface generation based on `backEnd.currentEntity->e.reType`.
- **Calls:** `RB_SurfaceSprite`, `RB_SurfaceBeam`, `RB_SurfaceRailCore`, `RB_SurfaceRailRings`, `RB_SurfaceLightningBolt`, `RB_SurfaceAxis`

### DoRailCore / DoRailDiscs
- **Purpose:** Internal helpers that generate the flat ribbon (`DoRailCore`) and rotating ring segments (`DoRailDiscs`) for rail gun trails.
- **Notes:** `DoRailCore` uses entity `shaderRGBA` with alpha-fade on one edge (×0.25).

## Control Flow Notes
This file is **back-end only** (called after `RB_BeginSurface`). The entry point is `rb_surfaceTable[surfaceType](surface)`, invoked from `RB_EndSurface`'s upstream caller in `tr_backend.c`. It runs within the render frame after sort-key ordering. It never touches front-end state.

## External Dependencies
- `tr_local.h` — all renderer types, `tess`, `backEnd`, `tr`, cvar externs
- `RB_BeginSurface`, `RB_EndSurface` — defined in `tr_cmds.c`/`tr_backend.c`
- `RB_SurfaceAnim` — defined in `tr_animation.c` (MD4)
- `GL_Bind`, `GL_State`, `qglBegin/End/Vertex/Color` — OpenGL wrappers; `RB_SurfaceBeam` and `RB_SurfaceAxis` bypass the tess buffer and issue immediate-mode GL directly
- `r_railWidth`, `r_railCoreWidth`, `r_railSegmentLength`, `r_lodCurveError` — cvars read at draw time
- `PerpendicularVector`, `RotatePointAroundVector`, `MakeNormalVectors`, `VectorNormalizeFast` — defined in `q_math.c`
