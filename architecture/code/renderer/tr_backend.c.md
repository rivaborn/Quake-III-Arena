# code/renderer/tr_backend.c

## File Purpose
This is the OpenGL render back end for Quake III Arena. It executes a command queue of render operations issued by the front end, manages OpenGL state transitions, and drives the actual draw calls for 3D surfaces and 2D UI elements.

## Core Responsibilities
- Maintain and cache OpenGL state (texture bindings, blend modes, depth, cull, alpha test) to minimize redundant API calls
- Execute the render command queue (`RC_SET_COLOR`, `RC_STRETCH_PIC`, `RC_DRAW_SURFS`, `RC_DRAW_BUFFER`, `RC_SWAP_BUFFERS`, `RC_SCREENSHOT`)
- Iterate sorted draw surfaces per-frame, batching by shader/fog/entity/dlight
- Set up per-entity model-view matrices and dynamic lighting transforms
- Support 2D orthographic rendering (UI, cinematics, stretch-pic)
- Handle SMP: optionally run the back end on a dedicated render thread

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `backEndState_t` | struct | All mutable back-end state: current entity, refdef, viewparms, 2D mode flag, color, perf counters |
| `backEndData_t` | struct | Double-buffered frame data: draw surfaces, dlights, entities, polys, render command list |
| `glstate_t` | struct | Cached OpenGL state (texture units, cull mode, blend bits) to avoid redundant GL calls |
| `drawSurf_t` | struct | A packed sort key + surface pointer; iterated in `RB_RenderDrawSurfList` |
| `renderCommand_t` | enum | Command IDs dispatched in `RB_ExecuteRenderCommands` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `backEndData` | `backEndData_t *[SMP_FRAMES]` | global | Double-buffered frame data for SMP support |
| `backEnd` | `backEndState_t` | global | Current back-end rendering state |
| `s_flipMatrix` | `float[16]` | static | Coordinate-system conversion matrix (game → OpenGL) used for portal clip plane setup |

## Key Functions / Methods

### GL_Bind
- **Signature:** `void GL_Bind( image_t *image )`
- **Purpose:** Bind a texture to the current TMU, skipping if already bound.
- **Inputs:** `image` — image to bind; NULL falls back to `tr.defaultImage`.
- **Outputs/Return:** None.
- **Side effects:** Updates `glState.currenttextures`, `image->frameUsed`, calls `qglBindTexture`.
- **Calls:** `qglBindTexture`, `ri.Printf`
- **Notes:** `r_nobind` redirects all binds to `tr.dlightImage` for performance profiling.

### GL_SelectTexture
- **Signature:** `void GL_SelectTexture( int unit )`
- **Purpose:** Switch the active TMU (0 or 1); errors on any other unit.
- **Side effects:** Updates `glState.currenttmu`, calls `qglActiveTextureARB` / `qglClientActiveTextureARB`.

### GL_Cull
- **Signature:** `void GL_Cull( int cullType )`
- **Purpose:** Set face culling mode, flipping front/back when rendering a mirror view.
- **Side effects:** Updates `glState.faceCulling`, calls `qglEnable/Disable/CullFace`.
- **Notes:** Mirror awareness via `backEnd.viewParms.isMirror`.

### GL_State
- **Signature:** `void GL_State( unsigned long stateBits )`
- **Purpose:** Bulk-apply a bitmask of GL state changes (blend, depth, alpha test, polygon mode). Only applies bits that differ from current state.
- **Side effects:** Updates `glState.glStateBits`; calls multiple `qgl*` functions.
- **Notes:** Central state-change bottleneck. All state is encoded in `GLS_*` bit flags.

### RB_BeginDrawingView
- **Signature:** `void RB_BeginDrawingView( void )`
- **Purpose:** Prepare GL for a new 3D view: set projection/viewport/scissor, clear buffers, handle hyperspace effect, set up portal clip plane.
- **Side effects:** Modifies `backEnd.projection2D`, `backEnd.skyRenderedThisView`, `glState.faceCulling`; calls `qglClear`, `qglClipPlane`, `qglEnable/Disable`.

### RB_RenderDrawSurfList
- **Signature:** `void RB_RenderDrawSurfList( drawSurf_t *drawSurfs, int numDrawSurfs )`
- **Purpose:** Core 3D draw loop. Iterates sorted surfaces, batches by shader/fog/entity, sets per-entity model-view matrix, handles depth-range hack for view models.
- **Side effects:** Calls `RB_BeginSurface`/`RB_EndSurface`, `R_RotateForEntity`, `R_TransformDlights`, `qglLoadMatrixf`, `qglDepthRange`, `RB_ShadowFinish`, `RB_RenderFlares`.
- **Notes:** Uses fast-path when consecutive surfaces share the same sort key.

### RB_SetGL2D
- **Signature:** `void RB_SetGL2D( void )`
- **Purpose:** Switch to orthographic 2D projection covering the full screen; disables cull face and clip plane.
- **Side effects:** Sets `backEnd.projection2D = qtrue`, updates `backEnd.refdef.time/floatTime`.

### RB_ExecuteRenderCommands
- **Signature:** `void RB_ExecuteRenderCommands( const void *data )`
- **Purpose:** Main dispatch loop; walks the render command buffer and calls the appropriate handler for each command.
- **Side effects:** Sets `backEnd.smpFrame`; writes `backEnd.pc.msec`.
- **Notes:** Can be called synchronously (no SMP) or from `RB_RenderThread`.

### RB_RenderThread
- **Signature:** `void RB_RenderThread( void )`
- **Purpose:** Entry point for the dedicated render thread (SMP mode). Sleeps via `GLimp_RendererSleep` until work arrives, then calls `RB_ExecuteRenderCommands`.
- **Side effects:** Toggles `renderThreadActive`.

### Notes (minor functions)
- `GL_TexEnv`, `GL_BindMultitexture`: thin cached wrappers around `qglTexEnvf` / multi-TMU bind.
- `RB_Hyperspace`: flashes a grey screen during predicted teleport.
- `SetViewportAndScissor`: uploads projection matrix and sets viewport/scissor from `viewParms`.
- `RE_StretchRaw` / `RE_UploadCinematic`: upload RGBA cinematic frames to `tr.scratchImage[client]` via `qglTexImage2D`/`qglTexSubImage2D`.
- `RB_ShowImages`: debug utility that tiles all loaded textures on screen.
- `RB_SetColor`, `RB_StretchPic`, `RB_DrawSurfs`, `RB_DrawBuffer`, `RB_SwapBuffers`: individual render-command handlers.

## Control Flow Notes
- **Frame path:** Front end submits commands to `backEndData[n]->commands`; `RB_ExecuteRenderCommands` processes them. `RC_DRAW_SURFS` → `RB_DrawSurfs` → `RB_RenderDrawSurfList` → per-surface `rb_surfaceTable[]` callbacks. `RC_SWAP_BUFFERS` → `RB_SwapBuffers` → `GLimp_EndFrame`.
- **SMP:** `RB_RenderThread` runs the back end asynchronously; `renderThreadActive` is a volatile flag monitored by the front end.
- **2D/3D switch:** `backEnd.projection2D` tracks mode; `RB_SetGL2D` switches to ortho, `RB_BeginDrawingView` switches to perspective.

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `rb_surfaceTable[]` — surface dispatch table (defined in `tr_surface.c`)
  - `tess` (`shaderCommands_t`) — tesselator globals
  - `tr` (`trGlobals_t`), `glConfig`, `glState` — renderer globals
  - `RB_BeginSurface`, `RB_EndSurface`, `RB_CheckOverflow` — tesselator (`tr_shade.c`)
  - `R_DecomposeSort`, `R_RotateForEntity`, `R_TransformDlights` — front-end math helpers
  - `RB_ShadowFinish`, `RB_RenderFlares`, `RB_TakeScreenshotCmd` — other back-end modules
  - `GLimp_*` — platform-specific GL window/thread layer
  - `ri` (`refimport_t`) — engine import table (memory, print, time)
