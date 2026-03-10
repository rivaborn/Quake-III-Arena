# Subsystem Overview

## Purpose
The `code/renderer` subsystem is the complete OpenGL-based rendering module for Quake III Arena (id Tech 3). It implements a two-phase front-end/back-end pipeline that traverses the BSP world, culls and sorts draw surfaces, and executes a queued command stream against OpenGL to produce each frame. It is loaded as a swappable DLL whose sole public entry point is `GetRefAPI`, which returns a `refexport_t` vtable to the engine.

---

## Key Files

| File | Role |
|---|---|
| `tr_local.h` | Master internal header; defines all renderer-private types, globals (`tr`, `backEnd`, `tess`), constants, GL state flags, and function prototypes |
| `tr_public.h` | Public ABI boundary; declares `refexport_t`, `refimport_t`, `GetRefAPI`, and `REF_API_VERSION` |
| `tr_init.c` | Renderer init/shutdown; registers all cvars, initializes OpenGL and all subsystems, exposes `GetRefAPI` |
| `tr_main.c` | Front-end control flow; view setup, frustum culling, draw-surface collection, sort, and back-end dispatch |
| `tr_backend.c` | Back-end; executes the render command queue, manages OpenGL state cache, drives draw calls |
| `tr_shade.c` | Surface shading; applies multi-pass shader programs to tessellated geometry via OpenGL |
| `tr_shader.c` | Shader management; parses `.shader` files, maintains shader registry, optimizes multi-pass shaders |
| `tr_bsp.c` | BSP map loader; converts all BSP lumps to renderer-ready structures, uploads lightmaps, builds the world tree |
| `tr_world.c` | Front-end world traversal; PVS/frustum culling of BSP leaves and surfaces, dlight BSP distribution |
| `tr_image.c` | Image management; loads BMP/PCX/TGA/JPEG, resamples/mipmaps, uploads to GPU, owns skin registration |
| `tr_model.c` | Model loading and registry; handles MD3 and MD4 formats, tag interpolation, model bounds |
| `tr_cmds.c` | Command buffer; double-buffers render commands, manages SMP render thread lifecycle |
| `tr_surface.c` | Back-end surface tessellation dispatch; converts all surface types into `tess` geometry |
| `tr_scene.c` | Scene submission API; accepts entities, dlights, and polys from the game; triggers `R_RenderView` |
| `tr_curve.c` | Bézier patch subdivision; converts control-point grids into `srfGridMesh_t` with LOD and normals |
| `tr_animation.c` | MD4 skeletal animation; bone interpolation and weighted vertex skinning into `tess` |
| `tr_mesh.c` | MD3 front-end; culling, LOD selection, fog membership, draw-surface submission |
| `tr_light.c` | Lighting; trilinear light-grid sampling for entities, dlight transform and bmodel intersection |
| `tr_shade_calc.c` | Shader calculations; waveform eval, vertex deformation, color/alpha/texcoord generation for `tess` |
| `tr_sky.c` | Sky/cloud rendering; cube-face clipping, sky box geometry, cloud projection, sun billboard |
| `tr_shadows.c` | Shadow rendering; stencil shadow volumes and planar projection shadows |
| `tr_flares.c` | Lens flare system; depth-buffer occlusion testing, intensity interpolation, screen-quad rendering |
| `tr_marks.c` | Decal projection; BSP traversal and polygon clipping to produce mark fragments |
| `tr_noise.c` | 4D value noise; seeded lattice init and trilinear interpolation for shader turbulence |
| `tr_font.c` | Font system; loads pre-rendered glyph atlases from disk (or rasterizes TrueType via FreeType 2) |
| `qgl.h` | QGL abstraction layer; platform-conditional `qgl*` function pointers or macros over OpenGL 1.x |
| `qgl_linked.h` | Compile-time `#define qgl* gl*` aliases for statically-linked platforms |

---

## Core Responsibilities

- **World visibility:** Walk the BSP tree each frame using PVS cluster data and frustum planes to mark visible leaves and cull surfaces before submission (`tr_world.c`, `tr_main.c`).
- **Surface sorting and batching:** Collect draw surfaces from world geometry, BSP brush models, MD3/MD4 entity models, and client polygons into a unified sort list keyed by shader, fog, entity, and dlight; flush the sorted list to the back-end command queue (`tr_main.c`, `tr_scene.c`).
- **Shader system:** Parse, cache, and optimize multi-pass `.shader` definitions; collapse two-pass combos into single multitexture passes; provide runtime shader remapping and implicit fallback shaders (`tr_shader.c`).
- **OpenGL command execution:** Execute a double-buffered render command queue against the OpenGL driver with a stateful cache (texture bindings, blend modes, depth/cull/alpha test) to minimize redundant state changes (`tr_backend.c`, `tr_cmds.c`).
- **Surface tessellation:** Convert every surface type (BSP faces, Bézier grids, triangle soups, MD3/MD4 meshes, sprites, rails, beams) into interleaved vertex/index data in the global `tess` (`shaderCommands_t`) buffer (`tr_surface.c`, `tr_animation.c`).
- **Texture and image management:** Load, resample, gamma-correct, mipmap, and upload all game textures; maintain an image hash-table cache; manage skins (`tr_image.c`).
- **Dynamic and static lighting:** Trilinearly sample the per-world light grid for entity ambient/directional lighting; transform and distribute dynamic lights through the BSP tree and per-surface (`tr_light.c`, `tr_world.c`).
- **Platform-agnostic GL dispatch:** Provide `qgl*`-prefixed wrappers over every OpenGL 1.x entry point — dynamic function pointers on Windows/Linux, compile-time `#define` aliases on statically-linked platforms (`qgl.h`, `qgl_linked.h`).

---

## Key Interfaces & Data Flow

**Exposed to other subsystems (`refexport_t` vtable, populated in `tr_init.c`):**
- Scene building: `RE_ClearScene`, `RE_AddRefEntityToScene`, `RE_AddPolyToScene`, `RE_AddLightToScene`
- Frame control: `RE_BeginFrame`, `RE_EndFrame`
- Resource registration: `RE_RegisterModel`, `RE_RegisterShader`, `RE_RegisterShaderNoMip`, `RE_RegisterSkin`, `RE_RegisterFont`
- World loading: `RE_LoadWorldMap`
- Utility queries: `RE_LerpTag`, `RE_ModelBounds`, `RE_GetEntityToken`, `RE_MarkFragments`, `RE_inPVS`
- Screenshot and render-to-texture: `RE_TakeVideoFrame`, `RE_StretchPic`

**Consumed from other subsystems (`refimport_t ri`, populated by the engine at `GetRefAPI` time):**
- **Memory:** `ri.Hunk_Alloc`, `ri.Hunk_AllocateTempMemory`, `ri.Hunk_FreeTempMemory`, `ri.Malloc`, `ri.Free`
- **Filesystem:** `ri.FS_ReadFile`, `ri.FS_FreeFile`, `ri.FS_ListFiles`
- **Cvar/cmd:** `ri.Cvar_Get`, `ri.Cvar_Set`, `ri.Cmd_AddCommand`, `ri.Cmd_ExecuteText`
- **Collision:** `ri.CM_DrawDebugSurface`, `ri.CM_ClusterPVS` (via `qcommon` CM module)
- **Cinematic:** `ri.CIN_RunCinematic`, `ri.CIN_UploadCinematic`, `ri.CIN_PlayCinematic`
- **Platform GL window layer (`GLimp_*`):** `GLimp_Init`, `GLimp_Shutdown`, `GLimp_SetGamma`, `GLimp_SpawnRenderThread`, `GLimp_FrontEndSleep`, `GLimp_WakeRenderer` — defined in `win32/win_glimp.c`, `unix/linux_glimp.c`, or `macosx/macosx_glimp.m`
- **Shared math/util:** `VectorMA`, `DotProduct`, `CrossProduct`, `PlaneFromPoints`, `RotatePointAroundVector`, `Q_rsqrt`, `Com_Memcpy`, `Com_Memset`, etc. — from `q_shared.c` / `q_math.c`

---

## Runtime Role

**Initialization (`RE_BeginFrame` / `GetRefAPI` call chain):**
1. Engine calls `GetRefAPI` at DLL load time; renderer populates and returns `refexport_t`, stores `refimport_t ri`.
2. `R_Register` registers all renderer cvars with the engine.
3. `InitOpenGL` calls `GLimp_Init` (platform GL window creation), sets default GL state, queries GL extensions.
4. Subsystems initialize in order: `R_InitImages` → `R_InitShaders` → `R_InitSkins` → `R_ModelInit` → `R_InitFreeType` → `R_NoiseInit` / `R_InitFogTable`.
5. `R_InitCommandBuffers` allocates `backEndData[0]` and (if SMP) `backEndData[1]`; optionally spawns the render thread.

**Per-frame (`RE_BeginFrame` → scene submission → `RE_EndFrame`):**
1. `RE_BeginFrame`: enqueues `RC_DRAW_BUFFER` and toggles the SMP frame.
2. Game/cgame calls `RE_AddRefEntityToScene`, `RE_AddPolyToScene`, `RE_AddLightToScene` to populate `backEndData` arrays.
3. `RE_RenderScene` (called from `RE_EndFrame` or directly): copies `refdef_t` into `tr.refdef`, calls `R_RenderView`.
4. `R_RenderView` (front end): `R_MarkLeaves` → `R_RecursiveWorldNode` → entity/model surface adders → `R_SortDrawSurfs` → `R_AddDrawSurfCmd`.
5. `RE_EndFrame`: enqueues `RC_SWAP_BUFFERS`, then calls `R_IssueRenderCommands` to hand the command buffer to the back end (same thread or SMP wake).
6. `RB_ExecuteRenderCommands` (back end): iterates the command queue; `RC_DRAW_SURFS` calls `RB_DrawSurfs`, which iterates sorted surfaces, calls `RB_BeginSurface` / `rb_surfaceTable[type]()` / `RB_EndSurface` per batch.
7. `RB_EndSurface` calls into `tr_shade.c` which evaluates shader stages, calls `tr_shade_calc.c` for vertex transforms/colors/texcoords, then issues `R_DrawElements` / `R_DrawStripElements` via QGL.

**Shutdown:**
- `RE_Shutdown` frees images, shaders, world data, models, the FreeType library, command buffers, and calls `GLimp_Shutdown`.

---

## Notable Implementation Details

- **Two-phase front/back-end with optional SMP:** The front end (scene traversal, surface sort) and back end (OpenGL command execution) are decoupled via a double-buffered render command queue (`backEndData[0/1]`). On SMP-capable platforms, the back end runs on a dedicated render thread (`GLimp_SpawnRenderThread`), with `GLimp_FrontEndSleep` / `GLimp_WakeRenderer` for synchronization (`tr_cmds.c`).
- **QGL indirection layer:** Every OpenGL call is issued through a `qgl*`-prefixed wrapper. On Windows and Linux these are runtime-resolved function pointers (populated via `GetProcAddress`/`dlsym` in `win_qgl.c`/`linux_qgl.c`). On statically-linked platforms the wrappers collapse to zero-overhead `#define` aliases (`qgl_linked.h`). On macOS, static-inline wrappers in `macosx_qgl.h` add optional per-call logging and error checking.
- **Shader sort key packing:** Draw surfaces are sorted by a packed integer key encoding shader index, fog volume, entity, and dlight bits, enabling the back end to batch consecutive surfaces sharing the same render state without explicit state diffing.
- **Overbright lighting and gamma correction:** Lightmap textures are color-shifted on upload by `r_mapOverBrightBits` to implement overbright lighting. A separate gamma/intensity lookup table (built in `tr_image.c`) is applied to all diffuse textures before GPU upload, with `GLimp_SetGamma` adjusting the hardware gamma ramp.
- **Bézier patch stitching:** `tr_curve.c` adaptively subdivides patches to a configurable error tolerance (`r_subdivisions`) and supports post-hoc column/row insertion (`R_GridInsertColumn`, `R_GridInsertRow`) to eliminate LOD T-junction cracks between adjacent patches of different subdivision levels.
- **Implicit shader fallback:** When a texture reference has no corresponding `.shader` block, `tr_shader.c` synthesizes a minimal default shader wrapping the raw image, ensuring all surfaces have a valid `shader_t` regardless of asset completeness.
- **Depth-buffer flare occlusion:** `tr_flares.c` reads individual pixels from the depth buffer via `qglReadPixels` after opaque geometry is drawn to determine per-flare visibility, then interpolates intensity over time to avoid instantaneous pop.
- **MD4 skeletal skinning is CPU-side:** Bone matrix interpolation and weighted vertex deformation for the MD4 format (`tr_animation.c`) are performed on the CPU each frame, writing final positions and normals directly into the `tess` buffer rather than using GPU skinning.
