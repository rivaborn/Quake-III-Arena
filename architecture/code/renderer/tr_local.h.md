# code/renderer/tr_local.h

## File Purpose
This is the primary internal header for the Quake III Arena renderer module. It defines all renderer-private data structures, global state, constants, and function prototypes used across the renderer's front-end and back-end subsystems. No external code outside the renderer should include this file.

## Core Responsibilities
- Define all renderer-internal types: shaders, surfaces, models, textures, lights, fog, world BSP structures
- Declare the two major global singletons: `tr` (front-end globals) and `backEnd` (back-end state)
- Define the `shaderCommands_t` tesselator (`tess`) used by the back-end to batch geometry
- Declare the render command queue types and SMP double-buffering structures
- Expose all internal function prototypes grouped by subsystem (shaders, world, lights, curves, skies, etc.)
- Declare all renderer cvars as `extern cvar_t *`
- Define GL state abstraction types (`glstate_t`, `GLS_*` bit flags)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `dlight_t` | struct | Dynamic point light: origin, color, radius, transformed position |
| `trRefEntity_t` | struct | Renderer-side entity augmenting `refEntity_t` with lighting state |
| `orientationr_t` | struct | Local coordinate frame: origin, axis, viewOrigin, modelMatrix |
| `image_t` | struct | Linked-list node for a GL texture: name, dimensions, texnum, wrap mode |
| `shader_t` | struct | Full shader definition: sort, stages, deforms, fog, sky, state machine links |
| `shaderStage_t` | struct | One rendering pass of a shader: texture bundles, blend/alpha/color gen |
| `textureBundle_t` | struct | Animated image array + tex coord gen + tex mod chain for one TMU |
| `trRefdef_t` | struct | Extended `refdef_t` used internally; holds entities, dlights, polys, drawsurfs |
| `viewParms_t` | struct | Per-view camera parameters: frustum, projection matrix, portal info |
| `world_t` | struct | Loaded BSP world: nodes, surfaces, fogs, lightgrid, vis data |
| `model_t` | struct | Loaded model: may be BSP brush model, MD3, or MD4 |
| `trGlobals_t` | struct | Master front-end state: all registered shaders, images, models, skins, function tables |
| `backEndState_t` | struct | Back-end per-frame state: current entity, view, 2D mode flag |
| `shaderCommands_t` | struct | Tesselator batch: vertex/index/color/texcoord arrays + active shader |
| `backEndData_t` | struct | SMP double-buffer: drawsurfs, dlights, entities, polys, render command list |
| `srfGridMesh_t` | struct | Bezier patch surface with LOD and dlight info |
| `srfSurfaceFace_t` | struct | Planar BSP face surface |
| `srfTriangles_t` | struct | Triangle soup surface (misc_model geometry) |
| `drawSurf_t` | struct | Packed sort key + surface type pointer for qsort |
| `msurface_t` / `mnode_t` | struct | BSP tree node/leaf/surface for world traversal |
| `renderCommandList_t` | struct | Fixed-size byte queue of typed render commands |
| `shaderState_t` | struct | Named state entry for stateful/cycling shader system |
| `performanceCounters_t` / `backEndCounters_t` | struct | Per-frame profiling counters |
| `glstate_t` | struct | Cached GL state: current textures, face culling, state bits |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tr` | `trGlobals_t` | global | All front-end renderer globals: registered assets, function tables |
| `backEnd` | `backEndState_t` | global | Back-end current frame/view/entity state |
| `glConfig` | `glconfig_t` | global | Hardware GL capabilities (persists across renderer re-init) |
| `glState` | `glstate_t` | global | Cached GL state (persists across renderer re-init) |
| `tess` | `shaderCommands_t` | global | Tesselator vertex/index batch for the current surface |
| `backEndData[SMP_FRAMES]` | `backEndData_t *` | global | SMP double-buffer; second may be NULL if SMP disabled |
| `renderCommandList` | `volatile renderCommandList_t *` | global | Active command list pointer for SMP handoff |
| `renderThreadActive` | `volatile qboolean` | global | SMP render thread liveness flag |
| `ri` | `refimport_t` | global | Engine import table (filesystem, memory, cvars, etc.) |
| `rb_surfaceTable` | `void(*)[SF_NUM_SURFACE_TYPES]` | global | Jump table: surface type enum → back-end draw function |
| `gl_filter_min/max` | `int` | global | Current GL texture filter modes |
| `max_polys` / `max_polyverts` | `int` | global | Runtime-configurable polygon limits |
| ~60 `r_*` cvar pointers | `cvar_t *` | global | All renderer console variables |

## Key Functions / Methods

### RB_BeginSurface / RB_EndSurface
- Signature: `void RB_BeginSurface(shader_t *shader, int fogNum)` / `void RB_EndSurface(void)`
- Purpose: Open/close a tesselator batch for a given shader. `EndSurface` flushes the `tess` buffer to GL.
- Side effects: Modifies `tess` global; `EndSurface` issues GL draw calls.

### RB_CheckOverflow
- Signature: `void RB_CheckOverflow(int verts, int indexes)`
- Purpose: Flush `tess` if adding `verts`/`indexes` would overflow the batch arrays.
- Notes: Used via `RB_CHECKOVERFLOW` macro before appending geometry.

### R_AddDrawSurf
- Signature: `void R_AddDrawSurf(surfaceType_t *surface, shader_t *shader, int fogIndex, int dlightMap)`
- Purpose: Pack a surface + shader + fog + dlight into a 32-bit sort key and append to `tr.refdef.drawSurfs`.
- Side effects: Writes to front-end drawsurf list.

### R_DecomposeSort
- Signature: `void R_DecomposeSort(unsigned sort, int *entityNum, shader_t **shader, int *fogNum, int *dlightMap)`
- Purpose: Unpack the 32-bit sort key back into its component indices.

### RE_RenderScene
- Signature: `void RE_RenderScene(const refdef_t *fd)`
- Purpose: Public entry point to render a complete scene from the given `refdef_t`; drives front-end cull/sort, then enqueues a `drawSurfsCommand_t`.
- Calls: `R_RenderView`, `R_AddDrawSurfCmd`

### RE_BeginFrame / RE_EndFrame
- Signature: `void RE_BeginFrame(stereoFrame_t stereoFrame)` / `void RE_EndFrame(int *frontEndMsec, int *backEndMsec)`
- Purpose: Frame lifecycle. `BeginFrame` sets up the draw buffer; `EndFrame` enqueues swap and signals the render thread (SMP).

### R_RenderView
- Signature: `void R_RenderView(viewParms_t *parms)`
- Purpose: Front-end per-view processing: traverse world BSP, cull entities/surfaces, sort draw surfaces.

### GL_State
- Signature: `void GL_State(unsigned long stateVector)`
- Purpose: Diff the `GLS_*` bitfield against cached `glState.glStateBits` and issue only the changed GL calls.
- Side effects: Modifies `glState`.

### RB_ExecuteRenderCommands
- Signature: `void RB_ExecuteRenderCommands(const void *data)`
- Purpose: Back-end command processor; iterates the `renderCommandList_t` byte stream dispatching typed commands.

### R_Init / RE_Shutdown
- Signature: `void R_Init(void)` / `void RE_Shutdown(qboolean destroyWindow)`
- Purpose: Full renderer initialization (cvars, images, shaders, models, fonts) and teardown.

### Notes
- `R_FindShader`, `R_GetShaderByHandle`, `R_RemapShader` manage the shader registry and state-machine remapping.
- `R_SubdividePatchToGrid`, `R_GridInsertColumn/Row` handle Bezier patch tesselation.
- `RB_CalcEnvironmentTexCoords`, `RB_CalcWaveColor`, etc. are per-stage texture coordinate and color calculators called during `RB_EndSurface`.

## Control Flow Notes
- **Init**: `R_Init` → registers cvars, loads images/shaders/fonts, builds function tables.
- **Frame**: `RE_BeginFrame` → client calls `RE_AddRefEntityToScene` / `RE_AddLightToScene` / `RE_RenderScene` → front-end culls/sorts → `R_AddDrawSurfCmd` enqueues command → `RE_EndFrame` flushes command list to back-end (or wakes render thread).
- **Back-end**: `RB_ExecuteRenderCommands` processes `RC_DRAW_SURFS` → iterates sorted `drawSurf_t` array → calls `RB_BeginSurface` / surface draw function (via `rb_surfaceTable`) / `RB_EndSurface` per batch.
- **SMP**: `backEndData[SMP_FRAMES]` double-buffers all scene data so front-end and back-end can run concurrently; `R_ToggleSmpFrame` alternates `tr.smpFrame`.

## External Dependencies
- `../game/q_shared.h` — `vec3_t`, `cplane_t`, `qboolean`, `cvar_t`, `refEntity_t`, etc.
- `../qcommon/qfiles.h` — `md3Header_t`, `md4Header_t`, `drawVert_t`, `dshader_t`, BSP lump types, `SHADER_MAX_VERTEXES`
- `../qcommon/qcommon.h` — `refimport_t`, memory allocators, filesystem, cvar/cmd APIs
- `tr_public.h` — `refexport_t`, `refimport_t`, `glconfig_t`, `stereoFrame_t` (from `cgame/tr_types.h`)
- `qgl.h` — `qgl*` function pointer wrappers for OpenGL
- `GLimp_*` functions — platform-specific GL window/thread management (defined in `win32/` or `unix/`)
- `SHADER_MAX_VERTEXES` / `SHADER_MAX_INDEXES` — defined in `qfiles.h`, constrain `shaderCommands_t` arrays
