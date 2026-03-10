# code/renderer/tr_init.c

## File Purpose
This is the renderer initialization and shutdown module for Quake III Arena's OpenGL renderer. It registers all renderer cvars, initializes the OpenGL subsystem and renderer subsystems (images, shaders, models), and exposes the renderer's public API via `GetRefAPI`.

## Core Responsibilities
- Declare and define all renderer `cvar_t*` globals used across the renderer module
- Register all renderer cvars with the engine cvar system in `R_Register`
- Initialize OpenGL via `InitOpenGL` (calls platform `GLimp_Init`, sets default GL state)
- Initialize renderer subsystems: images, shaders, skins, models, FreeType, function tables
- Allocate SMP-aware back-end data buffers (`backEndData[0/1]`)
- Handle screenshot capture (TGA and JPEG) via a render command queue
- Provide `GetRefAPI` — the DLL entry point that returns the `refexport_t` vtable to the engine

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `vidmode_t` | struct | Maps a mode index to a description string, pixel width/height, and pixel aspect ratio |
| `screenshotCommand_t` | struct (defined in tr_local.h) | Render command payload for deferred screenshot execution |
| `refexport_t` | struct (defined in tr_public.h) | Function pointer table returned to the engine by `GetRefAPI` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `glConfig` | `glconfig_t` | global | Hardware/driver capabilities and current video mode; shared across renderer |
| `glState` | `glstate_t` | global | Cached GL state bits to avoid redundant GL calls |
| `r_vidModes[]` | `vidmode_t[]` | file-static (effectively) | Table of predefined video resolutions (modes 0–11) |
| `s_numVidModes` | `int` | static | Count of entries in `r_vidModes` |
| `max_polys` | `int` | global | Effective poly cap (max of cvar value and `MAX_POLYS`) |
| `max_polyverts` | `int` | global | Effective polyvert cap (max of cvar value and `MAX_POLYVERTS`) |
| `qglMultiTexCoord2fARB` / `qglActiveTextureARB` / `qglClientActiveTextureARB` | function pointers | global | ARB multitexture extension entry points |
| `qglLockArraysEXT` / `qglUnlockArraysEXT` | function pointers | global | Compiled vertex array extension entry points |
| All `r_*` cvar pointers (~80) | `cvar_t *` | global | Renderer configuration exposed to console and other renderer files |

## Key Functions / Methods

### AssertCvarRange
- **Signature:** `static void AssertCvarRange(cvar_t *cv, float minVal, float maxVal, qboolean shouldBeIntegral)`
- **Purpose:** Clamps a cvar to a valid range and optionally enforces integer values; prints a warning and corrects the cvar if out of bounds.
- **Inputs:** cvar pointer, min, max, integral flag
- **Outputs/Return:** void; modifies cvar via `ri.Cvar_Set`
- **Side effects:** May call `ri.Printf` + `ri.Cvar_Set`
- **Calls:** `ri.Printf`, `ri.Cvar_Set`, `va`

### InitOpenGL
- **Signature:** `static void InitOpenGL(void)`
- **Purpose:** One-time OpenGL subsystem init: calls `GLimp_Init` only if not already initialized (`glConfig.vidWidth == 0`), queries `GL_MAX_TEXTURE_SIZE`, initializes command buffers, and sets default GL state.
- **Side effects:** Populates `glConfig.maxTextureSize`; calls `R_InitCommandBuffers`, `GfxInfo_f`, `GL_SetDefaultState`
- **Calls:** `GLimp_Init`, `qglGetIntegerv`, `R_InitCommandBuffers`, `GfxInfo_f`, `GL_SetDefaultState`
- **Notes:** Guard on `vidWidth == 0` prevents re-init on renderer restarts that reuse the existing window.

### GL_CheckErrors
- **Signature:** `void GL_CheckErrors(void)`
- **Purpose:** Queries `glGetError` and calls `ri.Error(ERR_FATAL)` on any GL error, unless `r_ignoreGLErrors` is set.
- **Side effects:** Can terminate the engine
- **Calls:** `qglGetError`, `ri.Error`, `Com_sprintf`

### R_GetModeInfo
- **Signature:** `qboolean R_GetModeInfo(int *width, int *height, float *windowAspect, int mode)`
- **Purpose:** Resolves a video mode index to pixel dimensions and aspect ratio; mode `-1` uses custom cvars.
- **Inputs:** Output pointers for width/height/aspect, mode index
- **Outputs/Return:** `qtrue` on success, `qfalse` if mode out of range
- **Calls:** Reads `r_customwidth`, `r_customheight`, `r_customaspect`

### RB_TakeScreenshot
- **Signature:** `void RB_TakeScreenshot(int x, int y, int width, int height, char *fileName)`
- **Purpose:** Captures the framebuffer as a TGA file; handles RGB swap and optional gamma correction.
- **Side effects:** Allocates/frees temp memory; writes file via `ri.FS_WriteFile`
- **Calls:** `ri.Hunk_AllocateTempMemory`, `qglReadPixels`, `R_GammaCorrect`, `ri.FS_WriteFile`, `ri.Hunk_FreeTempMemory`

### RB_TakeScreenshotJPEG
- **Signature:** `void RB_TakeScreenshotJPEG(int x, int y, int width, int height, char *fileName)`
- **Purpose:** JPEG variant; reads RGBA framebuffer, gamma-corrects, then calls `SaveJPG` at quality 95.
- **Side effects:** Same as above plus calls `SaveJPG`
- **Notes:** The `ri.FS_WriteFile(..., 1)` call is a path-creation stub (writes 1 byte to ensure the directory exists before `SaveJPG` writes the real file).

### RB_TakeScreenshotCmd
- **Signature:** `const void *RB_TakeScreenshotCmd(const void *data)`
- **Purpose:** Render-thread command handler; dispatches to TGA or JPEG screenshot based on `cmd->jpeg`.
- **Outputs/Return:** Pointer to next command in the render command buffer (`cmd + 1`)

### R_TakeScreenshot
- **Signature:** `void R_TakeScreenshot(int x, int y, int width, int height, char *name, qboolean jpeg)`
- **Purpose:** Front-end: enqueues a `screenshotCommand_t` into the render command buffer for deferred execution on the render thread.
- **Side effects:** Uses a `static char fileName[]` — only one pending screenshot filename at a time.
- **Calls:** `R_GetCommandBuffer`

### GL_SetDefaultState
- **Signature:** `void GL_SetDefaultState(void)`
- **Purpose:** Establishes the known-good baseline OpenGL state (depth func, cull face, texturing, blend, scissor) and syncs `glState.glStateBits`.
- **Side effects:** Issues many direct GL calls; modifies `glState`
- **Calls:** `qglClearDepth`, `qglCullFace`, `GL_SelectTexture`, `GL_TextureMode`, `GL_TexEnv`, `qglEnable/Disable`, `qglShadeModel`, `qglDepthFunc`, etc.

### R_Register
- **Signature:** `void R_Register(void)`
- **Purpose:** Registers all ~80 renderer cvars with the engine and adds console commands (`screenshot`, `screenshotJPEG`, `gfxinfo`, `imagelist`, etc.).
- **Side effects:** Populates all `r_*` global cvar pointers; adds commands via `ri.Cmd_AddCommand`
- **Notes:** Linux-specific defaults differ for `r_stencilbits` (0) and `r_ext_texture_env_add` (0, disabled due to driver bugs). macOS defaults `r_gamma` to 1.2. SMP defaults to enabled on Mac/Linux if `Sys_ProcessorCount() > 1`.

### R_Init
- **Signature:** `void R_Init(void)`
- **Purpose:** Top-level renderer initialization called once at startup. Zeroes global state, builds waveform function tables (`sinTable`, `squareTable`, etc.), allocates `backEndData` SMP buffers, then calls all subsystem inits in order.
- **Side effects:** Allocates hunk memory; modifies `tr`, `backEnd`, `tess`; calls `InitOpenGL`, `R_InitImages`, `R_InitShaders`, `R_InitSkins`, `R_ModelInit`, `R_InitFreeType`
- **Calls:** `R_InitFogTable`, `R_NoiseInit`, `R_Register`, `ri.Hunk_Alloc`, `R_ToggleSmpFrame`, `InitOpenGL`, `R_InitImages`, `R_InitShaders`, `R_InitSkins`, `R_ModelInit`, `R_InitFreeType`

### RE_Shutdown
- **Signature:** `void RE_Shutdown(qboolean destroyWindow)`
- **Purpose:** Tears down the renderer: removes console commands, syncs/shuts down command buffers, deletes textures, shuts down FreeType, and optionally destroys the GL window.
- **Side effects:** Sets `tr.registered = qfalse`; calls `GLimp_Shutdown` if `destroyWindow`

### GetRefAPI
- **Signature:** `refexport_t *GetRefAPI(int apiVersion, refimport_t *rimp)`
- **Purpose:** DLL entry point. Validates API version, copies the `refimport_t` into the global `ri`, populates and returns a static `refexport_t` vtable.
- **Outputs/Return:** Pointer to static `refexport_t`, or `NULL` on version mismatch
- **Notes:** All renderer entry points visible to the engine (`RE_*`, `R_*`) are wired here.

## Control Flow Notes
- **Init:** Engine calls `GetRefAPI` → sets up `ri` import table → engine calls `RE_BeginRegistration` → calls `R_Init` → `R_Register` (cvars) → `InitOpenGL` → subsystem inits.
- **Frame:** Not directly involved in per-frame rendering; screenshot commands enqueued here are consumed by `RB_ExecuteRenderCommands` on the render thread.
- **Shutdown:** Engine calls `RE_Shutdown` → cleans up commands/textures → optionally destroys GL window.

## External Dependencies
- `tr_local.h` — all renderer-internal types, globals, and function declarations
- `GLimp_Init` / `GLimp_Shutdown` — platform-specific GL window creation (defined in `win_glimp.c` / `linux_glimp.c` / `macosx_glimp.m`)
- `SaveJPG` — JPEG encoder (defined in `tr_image.c`)
- `ri` (`refimport_t`) — engine callbacks for cvars, commands, file I/O, memory, printing (defined elsewhere, imported via `GetRefAPI`)
- `R_InitImages`, `R_InitShaders`, `R_InitSkins`, `R_ModelInit`, `R_InitFreeType` — defined in their respective `tr_*.c` files
- `R_InitCommandBuffers`, `R_ToggleSmpFrame` — defined in `tr_cmds.c`
- `R_InitFogTable`, `R_NoiseInit` — defined in `tr_noise.c` / fog subsystem
