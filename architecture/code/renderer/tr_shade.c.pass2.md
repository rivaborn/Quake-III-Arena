# code/renderer/tr_shade.c — Enhanced Analysis

## Architectural Role

`tr_shade.c` is the **execution core of the renderer's back-end shading pipeline**. It sits at the precise boundary between the CPU-side shader evaluation (colors, texcoords, deforms) and the OpenGL draw submission. The front-end (`tr_world.c`, `tr_scene.c`, `tr_main.c`) culls and sorts surfaces; `tr_backend.c` dequeues render commands and calls `RB_BeginSurface`/`RB_EndSurface`; this file is what actually evaluates shader stages and issues `glDraw*` calls. The global `tess` (`shaderCommands_t`) buffer defined here is the single shared accumulation point for all surface tessellation across the entire back-end — every surface type (BSP faces, MD3 meshes, Bézier patches, sprites) funnels geometry through it before this file flushes it to GL.

The three `RB_StageIterator*` functions implement a **strategy pattern** chosen at shader-load time (`tr_shader.c` writes `shader->optimalStageIteratorFunc`), allowing the common-case world surfaces and vertex-lit models to skip the full generic multi-pass evaluation loop entirely.

## Key Cross-References

### Incoming (who depends on this file)

- **`tr_backend.c`**: calls `RB_BeginSurface` / `RB_EndSurface` for every surface batch in the render command queue; this is the primary and only legitimate caller.
- **`tr_surface.c`** (via `rb_surfaceTable` dispatch), **`tr_mesh.c`**, **`tr_animation.c`**, **`tr_curve.c`**, **`tr_bsp.c`**, **`tr_sky.c`**, **`tr_marks.c`**: all write into `tess.xyz`, `tess.indexes`, `tess.normal`, etc. between `RB_BeginSurface` and `RB_EndSurface` calls. The `tess` global is effectively the shared contract between this file and all surface tessellation code.
- **`tr_shader.c`**: reads `shader->optimalStageIteratorFunc` which is set to point at one of the three `RB_StageIterator*` functions during shader finalization.
- **`tr_flares.c`**, **`tr_shadows.c`**: call `RB_BeginSurface`/`RB_EndSurface` for their own geometry passes; `RB_ShadowTessEnd` (called from `RB_EndSurface`) is defined in `tr_shadows.c`.

### Outgoing (what this file depends on)

- **`tr_shade_calc.c`**: all `RB_Calc*` helpers (`RB_CalcDiffuseColor`, `RB_CalcWaveColor`, `RB_CalcScrollTexCoords`, `RB_CalcEnvironmentTexCoords`, `RB_CalcTurbulentTexCoords`, etc.) and `RB_DeformTessGeometry` — the bulk of per-vertex math lives there.
- **`tr_light.c`**: populates `backEnd.refdef.dlights[]` and sets `tess.dlightBits` per surface; `ProjectDlightTexture` here consumes both.
- **`tr_init.c` / `tr_main.c`**: `GL_Bind`, `GL_State`, `GL_Cull`, `GL_SelectTexture`, `GL_TexEnv` — the stateful GL wrapper layer.
- **`tr_image.c`**: `tr.whiteImage` and the image array used by `R_BindAnimatedImage`.
- **Platform layer**: `GLimp_LogComment` for optional GL call logging; `ri.CIN_*` for video texture updates; `ri.Error` for fatal overflow detection.
- **Cvars read**: `r_primitives`, `r_lightmap`, `r_showtris`, `r_shownormals`, `r_offsetFactor`, `r_offsetUnits`, `r_debugSort`, `r_vertexLight`.

## Design Patterns & Rationale

- **Strategy via function pointer** (`currentStageIteratorFunc`): The three iterators — Generic, VertexLitTexture, LightmappedMultitexture — represent the three dominant surface archetypes of the era. Choosing at shader-load time avoids a per-surface branch in the hot path. This was important when multitexture hardware was new and state transitions were expensive.
- **Batch accumulation with explicit flush**: The `tess` buffer separates geometry assembly (surface tessellators) from GL submission (this file). The overflow guard in `RB_EndSurface` (checking sentinel index values) handles cases where a surface exceeds buffer capacity mid-batch — a pragmatic safety valve rather than a resize.
- **Runtime-selectable primitive mode** (`r_primitives` cvar): The `R_DrawElements` dispatch between strip detection and `glDrawElements` reflects the GPU performance landscape of 1998–1999. Strip topology was favored for cache efficiency before transform-and-lighting hardware; with CVA (`qglLockArraysEXT`), indexed triangles won. The `primitives==3` discrete element path was a conformance harness, not a production path.
- **Whole-pass dlight and fog**: Rather than integrating dynamic lighting into the per-stage loop, `ProjectDlightTexture` and `RB_FogPass` are unconditional post-passes. This simplifies stage iteration at the cost of an extra draw call per affected light. Modern renderers would fold this into a lighting pre-pass or deferred buffer.

## Data Flow Through This File

```
tr_backend.c
  └─ RB_BeginSurface(shader, fogNum)
       → resets tess counts, installs shader/iterator/shaderTime

rb_surfaceTable[surf->surfaceType](surf)   [tr_surface.c etc.]
       → appends to tess.xyz / tess.indexes / tess.normal

  └─ RB_EndSurface()
       → overflow check
       → currentStageIteratorFunc()  [one of three below]
            └─ RB_DeformTessGeometry()        [tr_shade_calc.c]
            └─ ComputeColors(stage)           [reads rgbGen/alphaGen → tess.svars.colors]
            └─ ComputeTexCoords(stage)        [reads tcGen/texMods → tess.svars.texcoords]
            └─ R_DrawElements(...)            [→ glDrawElements or strip path]
            └─ ProjectDlightTexture()         [additive pass per dlight bit]
            └─ RB_FogPass()                   [blended fog pass if fogNum set]
       → backEnd.pc counters updated
```

Key state transitions: `tess.dlightBits` is OR'd in by surface tessellators (from `tr_light.c` assignment) and consumed here. `tess.shaderTime` is computed once in `RB_BeginSurface` and used across all `ComputeColors`/`ComputeTexCoords` calls so waveform animations stay coherent within a batch.

## Learning Notes

- **Era-specific multi-pass rendering**: Everything Q3 does in CPU-driven multi-pass GL calls (base texture + lightmap + dlight + fog = 3–4 draw calls per surface) a modern engine folds into a single fragment shader invocation. Studying these passes reveals exactly what GLSL was invented to replace.
- **The lightmap architecture**: The `RB_StageIteratorLightmappedMultitexture` path encodes Q3's core rendering contract: world surfaces are pre-lit into baked lightmaps (TMU1) combined with diffuse (TMU0) via `GL_MODULATE`. This is why dynamic world re-lighting is architecturally absent — it would require rebuilding lightmaps or a deferred path that didn't exist.
- **CVA as the "GPU buffer" of 1999**: `qglLockArraysEXT` (Compiled Vertex Arrays) is the era's equivalent of a vertex buffer object — it hints to the driver to cache the vertex data on the GPU side. The `setArraysOnce` flag optimizes CVA usage by binding arrays once before the lock rather than per-stage.
- **AltiVec in `ProjectDlightTexture`**: The `#if idppc_altivec` SIMD path demonstrates that id Software hand-wrote platform-specific SIMD at a time before auto-vectorization was reliable. The `vec_ld(11, floatColor)` idiom is the AltiVec unaligned load pattern — loading two aligned vectors and permuting to extract an unaligned float3.
- **No scene graph, no ECS**: Surfaces arrive as raw geometry arrays. There is no spatial hierarchy, component system, or material abstraction above `shaderStage_t`. This flat, cache-friendly structure was deliberate and influential on subsequent engines.

## Potential Issues

- **`tess` is not thread-safe**: The SMP renderer (`tr_smp.c`) runs back-end GL submission on a second thread. All callers of `RB_BeginSurface`/`RB_EndSurface` must be exclusively on the back-end thread; any accidental front-end access to `tess` during an SMP frame would silently corrupt geometry.
- **`c_vertexes` / `c_begins` never reset**: These static counters accumulate since process start. They are used to assess strip efficiency but give only lifetime averages, not per-frame statistics.
- **Overflow sentinel is one-sided**: `RB_EndSurface` checks `tess.indexes[SHADER_MAX_INDEXES-1]` for a canary value but only calls `ri.Error` if it detects overflow; it does not prevent the overwrite from already having occurred, making this a detection mechanism rather than a guard.
