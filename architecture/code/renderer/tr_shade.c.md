# code/renderer/tr_shade.c

## File Purpose
This is the renderer back end's surface shading module. It applies shader programs (multi-stage, multi-pass) to tessellated surface geometry stored in the global `tess` struct, dispatching to OpenGL draw calls with appropriate texture, color, and blend state.

## Core Responsibilities
- Initialize and finalize per-surface tessellation batches (`RB_BeginSurface`, `RB_EndSurface`)
- Compute per-vertex colors (`ComputeColors`) and texture coordinates (`ComputeTexCoords`) for each shader stage
- Dispatch geometry to OpenGL via triangle strips or indexed triangles (`R_DrawElements`, `R_DrawStripElements`)
- Handle multi-pass rendering: generic stages, vertex-lit, lightmapped multitexture
- Apply dynamic light projections as additive/modulate passes (`ProjectDlightTexture`)
- Apply fog blending pass (`RB_FogPass`)
- Support debug visualization of triangle wireframes and vertex normals

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `shaderCommands_t` | struct (typedef) | Global tess buffer: holds indexes, XYZ, normals, texcoords, colors, shader ref, and stage iterator fn pointer |
| `stageVars_t` | struct (typedef) | Per-frame computed colors and texcoords written out per shader stage |
| `shaderStage_t` | struct (typedef) | One stage of a shader: blend state, texture bundles, rgbGen/alphaGen config |
| `textureBundle_t` | struct (typedef) | Texture image(s) + animation speed + tcGen mode + texmods for one TMU |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tess` | `shaderCommands_t` | global | Central tessellator accumulation buffer shared across entire back end |
| `setArraysOnce` | `static qboolean` | static | Tracks whether GL client arrays were bound before CVA lock (single-pass optimization) |
| `c_vertexes` | `static int` | static | Debug counter: total vertexes emitted by strip renderer |
| `c_begins` | `static int` | static | Debug counter: number of GL_TRIANGLE_STRIP begin calls |

## Key Functions / Methods

### RB_BeginSurface
- **Signature:** `void RB_BeginSurface(shader_t *shader, int fogNum)`
- **Purpose:** Resets the tess struct to start accumulating a new surface batch.
- **Inputs:** `shader` — resolved shader (or remapped), `fogNum` — fog region index
- **Outputs/Return:** void; writes into global `tess`
- **Side effects:** Modifies `tess.*` fields (counts, shader ptr, stage iterator fn, shaderTime)
- **Calls:** None directly; reads `backEnd.refdef.floatTime`
- **Notes:** Handles shader time clamping. Must be called before any geometry is added to `tess`.

### RB_EndSurface
- **Signature:** `void RB_EndSurface(void)`
- **Purpose:** Finalizes a surface batch; invokes the shader's stage iterator, debug draws, and increments perf counters.
- **Inputs:** global `tess`
- **Outputs/Return:** void
- **Side effects:** Calls `tess.currentStageIteratorFunc()`, updates `backEnd.pc`, resets `tess.numIndexes`
- **Calls:** `RB_ShadowTessEnd`, `tess.currentStageIteratorFunc`, `DrawTris`, `DrawNormals`, `GLimp_LogComment`
- **Notes:** Overflow guard checks sentinel values at `SHADER_MAX_INDEXES-1` and `SHADER_MAX_VERTEXES-1`.

### RB_StageIteratorGeneric
- **Signature:** `void RB_StageIteratorGeneric(void)`
- **Purpose:** General-purpose multi-pass shader executor. Handles arbitrary stage count, CVA locking, polygon offset, dlight, and fog passes.
- **Inputs:** global `tess`
- **Outputs/Return:** void
- **Side effects:** Issues OpenGL state changes, vertex array setup, CVA lock/unlock, calls `RB_IterateStagesGeneric`, `ProjectDlightTexture`, `RB_FogPass`
- **Calls:** `RB_DeformTessGeometry`, `GL_Cull`, `qglLockArraysEXT`, `RB_IterateStagesGeneric`, `ProjectDlightTexture`, `RB_FogPass`, `qglUnlockArraysEXT`

### RB_StageIteratorVertexLitTexture
- **Signature:** `void RB_StageIteratorVertexLitTexture(void)`
- **Purpose:** Optimized path for single-stage vertex-lit surfaces (pre-lit models, no lightmap).
- **Side effects:** Calls `RB_CalcDiffuseColor`, binds stage 0, issues draw, handles dlights and fog
- **Calls:** `RB_CalcDiffuseColor`, `GL_Cull`, `R_BindAnimatedImage`, `R_DrawElements`, `ProjectDlightTexture`, `RB_FogPass`

### RB_StageIteratorLightmappedMultitexture
- **Signature:** `void RB_StageIteratorLightmappedMultitexture(void)`
- **Purpose:** Optimized path for world surfaces using simultaneous diffuse + lightmap multitexture (TMU 0 + TMU 1).
- **Calls:** `GL_SelectTexture`, `R_BindAnimatedImage`, `R_DrawElements`, `ProjectDlightTexture`, `RB_FogPass`
- **Notes:** Has a compiled-out `REPLACE_MODE` path using `GL_FLAT` shading.

### ProjectDlightTexture
- **Signature:** `static void ProjectDlightTexture(void)`
- **Purpose:** Adds a projected dynamic light texture pass over surfaces flagged in `tess.dlightBits`.
- **Inputs:** `backEnd.refdef.dlights`, `tess` geometry
- **Side effects:** Writes temp `texCoordsArray`/`colorArray`, issues additive or modulate blend draw per light
- **Calls:** `R_DrawElements`, `GL_Bind`, `GL_State`
- **Notes:** Has AltiVec SIMD path (`#if idppc_altivec`) for PPC platforms.

### ComputeColors
- **Signature:** `static void ComputeColors(shaderStage_t *pStage)`
- **Purpose:** Evaluates `rgbGen` and `alphaGen` directives into `tess.svars.colors`.
- **Calls:** Various `RB_Calc*` helpers (`RB_CalcDiffuseColor`, `RB_CalcWaveColor`, `RB_CalcSpecularAlpha`, fog modulate functions, etc.)

### ComputeTexCoords
- **Signature:** `static void ComputeTexCoords(shaderStage_t *pStage)`
- **Purpose:** Evaluates `tcGen` and all `texMod` operations into `tess.svars.texcoords[b]`.
- **Calls:** `RB_CalcFogTexCoords`, `RB_CalcEnvironmentTexCoords`, `RB_CalcTurbulentTexCoords`, `RB_CalcScrollTexCoords`, etc.

### R_DrawElements
- **Signature:** `static void R_DrawElements(int numIndexes, const glIndex_t *indexes)`
- **Purpose:** Routes to `glDrawElements` or `R_DrawStripElements` depending on `r_primitives` cvar.
- **Notes:** `r_primitives==2` uses straight triangles (preferred with CVA); `1` uses strip detection with `qglArrayElement`; `3` is conformance-testing mode.

## Control Flow Notes
- Called from the **render back end** only, after front-end culling and sorting.
- Per-frame flow: `RB_BeginSurface` → geometry accumulation (from `rb_surfaceTable` callbacks) → `RB_EndSurface` → `currentStageIteratorFunc()` (one of the three `RB_StageIterator*` functions) → optional dlight pass → optional fog pass.
- `tess.currentStageIteratorFunc` is set in `RB_BeginSurface` from `shader->optimalStageIteratorFunc`, chosen at shader load time.

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `backEnd`, `tr`, `glConfig`, `glState` — renderer globals
  - `r_primitives`, `r_logFile`, `r_lightmap`, `r_showtris`, `r_shownormals`, `r_vertexLight`, `r_uiFullScreen`, `r_offsetFactor`, `r_offsetUnits`, `r_debugSort` — cvars
  - `RB_DeformTessGeometry`, all `RB_Calc*` functions — defined in `tr_shade_calc.c`
  - `RB_ShadowTessEnd` — defined in `tr_shadows.c`
  - `GL_Bind`, `GL_State`, `GL_Cull`, `GL_SelectTexture`, `GL_TexEnv` — `tr_init.c` / `tr_main.c`
  - `GLimp_LogComment` — platform-specific implementation
  - `ri.CIN_RunCinematic`, `ri.CIN_UploadCinematic`, `ri.Error` — engine import table
