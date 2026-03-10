# code/renderer/tr_sky.c — Enhanced Analysis

## Architectural Role

This file implements **sky rendering** as a specialized surface type within the renderer's two-phase front-end/back-end architecture. The sky is not rendered as ordinary BSP geometry; instead, it uses a hybrid approach: the outer skybox is drawn immediately via OpenGL immediate-mode commands, while cloud layers are batched into the tessellator (`tess`) for later dispatch through the generic stage iterator. This special-case handling is necessary because sky geometry is view-relative (centered on the camera at `backEnd.viewParms.or.origin`) and must be clipped against a fixed six-plane convex frustum, decoupled from world-space BSP rendering.

## Key Cross-References

### Incoming (who depends on this file)
- **`RB_StageIteratorSky`** is registered as `shader_t::optimalStageIteratorFunc` during shader parsing (implicitly, in `tr_shader.c`). The surface dispatch system calls this when rendering a sky shader surface.
- **`R_InitSkyTexCoords`** is called from the shader parsing system (implicitly in `tr_shader.c`) when a sky shader with `skyparms` is parsed.
- **`RB_DrawSun`** is called from the main scene render loop (implicitly in `tr_main.c`) if `backEnd.skyRenderedThisView` is true (set by `RB_StageIteratorSky`).

### Outgoing (what this file depends on)
- **Renderer subsystem services:** `GL_Bind` (texture binding), `GL_State` (rasterizer state), `RB_BeginSurface`/`RB_EndSurface` (tessellator interface), `RB_StageIteratorGeneric` (cloud stage rendering via the standard pipeline).
- **OpenGL immediate-mode wrappers:** `qglBegin`, `qglEnd`, `qglVertex3fv`, `qglTexCoord2fv`, `qglColor3f`, `qglDepthRange`, `qglPushMatrix`, `qglPopMatrix`, `qglLoadMatrixf`, `qglTranslatef` (from `qgl.h`).
- **Math utilities:** `VectorAdd`, `VectorSubtract`, `VectorCopy`, `DotProduct`, `CrossProduct`, `VectorScale`, `VectorNormalize`, `PerpendicularVector`, `Q_acos` (from `q_math.c` and `math_vector.h`).
- **Global state:** `tess` (tessellator), `backEnd` (frame-level rendering state), `tr` (renderer globals including `sunDirection`, `sunTexture`), `myftol` (platform-specific float-to-int), `ri` (engine import interface).

## Design Patterns & Rationale

**1. Precomputation + Runtime Indirection**
- `R_InitSkyTexCoords` (called once per shader parse) fills `s_cloudTexCoords[6][9][9][2]` using expensive spherical projection math (`DotProduct`, `sqrt`, `VectorNormalize`).
- At runtime, `FillCloudySkySide` reads these precomputed values, avoiding per-vertex computation and enabling **fast cloud layer redraws** across multiple rendering passes (e.g., different lightmaps or overlays on the cloud).
- This decouples the cost (shader parse time, amortized across a level) from the critical path (per-frame render).

**2. Subdivided Mesh with Clipping Bounds**
- `RB_ClipSkyPolygons` uses **convex polygon clipping** against six planes (axis-aligned cube faces), recursively subdividing the input BSP triangles.
- The accumulated `sky_mins[2][6]` / `sky_maxs[2][6]` describe which **subregions of each cube face** are actually visible—a form of **viewport culling** specialized for sky.
- `DrawSkyBox` then only generates and renders geometry for those visible subregions, reducing vertex count and saving GPU work compared to always rendering all six full faces.
- This is a classic **visible-region optimization** for large, static surfaces.

**3. Immediate-Mode for Sky Box, Retained-Mode for Clouds**
- The outer box is rendered with `qglBegin(GL_TRIANGLE_STRIP)` / `qglEnd()` in `DrawSkySide`, a per-frame draw.
- Clouds are batched into `tess` (retained-mode geometry) in `FillCloudySkySide`, then flushed through `RB_StageIteratorGeneric` like any other shader stage.
- This hybrid design likely reflects **pragmatic optimization**: the sky box rarely needs to be dynamic (no animation, lighting, or vertex modification), so immediate-mode is acceptable; clouds, however, benefit from batching with other transparent surfaces (sorting, depth testing, etc.).

**4. View-Relative Coordinate Transformation**
- `MakeSkyVec` converts cube-face ST coordinates `[-1, 1]` to **world-space 3D points** at distance `backEnd.viewParms.zFar / 1.75` from the camera origin.
- All sky geometry is then offset by `backEnd.viewParms.or.origin` in `FillCloudySkySide` (` VectorAdd(..., backEnd.viewParms.or.origin, tess.xyz[...])`) and `DrawSkyBox` (implicit via `qglVertex3fv` of precomputed `s_skyPoints`).
- This ensures the sky **always moves with the camera** but appears infinitely far away (no parallax).

**5. Cvar-Driven Rendering Modes**
- `r_fastsky` (checked in `RB_StageIteratorSky`) skips all sky rendering if true.
- `r_drawSun` (checked in `RB_DrawSun`) controls whether the sun quad is rendered.
- `r_showsky` (inferred from patterns) likely enables debug visualization.
- These allow **cheap toggles** for profiling and art direction.

## Data Flow Through This File

**Per-Frame Sky Render Sequence:**

```
1. RB_StageIteratorSky (entry point from surface dispatch)
   ├─ RB_ClipSkyPolygons
   │  ├─ ClearSkyBox          [reset sky_mins/sky_maxs globals]
   │  ├─ ClipSkyPolygon (6 recursive stages, per input triangle)
   │  │  └─ AddSkyPolygon     [accumulate bounds on each cube face]
   │  └─ [sky_mins/sky_maxs now describe visible regions per face]
   │
   ├─ DrawSkyBox (shader_t *shader)
   │  ├─ MakeSkyVec (per subdivision grid point)
   │  │  └─ [compute 3D world-space point + clamped UV]
   │  ├─ DrawSkySide (per visible region, immediate-mode GL)
   │  │  ├─ GL_Bind(outerbox texture)
   │  │  ├─ qglBegin(GL_TRIANGLE_STRIP)
   │  │  └─ qglVertex3fv / qglTexCoord2fv (per vertex)
   │  └─ [outer box now on-screen]
   │
   ├─ R_BuildCloudData
   │  ├─ FillCloudBox (per shader stage with clouds)
   │  │  └─ FillCloudySkySide (per cube face)
   │  │     ├─ [read precomputed s_cloudTexCoords]
   │  │     ├─ VectorAdd(..., backEnd.viewParms.or.origin)
   │  │     └─ [accumulate into tess.xyz, tess.texCoords, tess.indexes]
   │  └─ [cloud geometry now in tess, ready for batch flush]
   │
   └─ RB_StageIteratorGeneric
      └─ [render tess (clouds + any other geometry) with state machine]

2. (Later, after main render) RB_DrawSun (if backEnd.skyRenderedThisView)
   ├─ Compute billboard quad perpendicular to tr.sunDirection
   ├─ RB_BeginSurface / RB_EndSurface
   └─ [sun quad drawn at far clip plane]
```

**Key State Mutations:**
- `sky_mins[2][6]`, `sky_maxs[2][6]`: Accumulated via `AddSkyPolygon`; used to determine visible regions.
- `s_skyPoints[9][9]`, `s_skyTexCoords[9][9][2]`: Regenerated per-frame in `DrawSkyBox` as scratch buffers.
- `tess.numVertexes`, `tess.numIndexes`, `tess.xyz`, `tess.texCoords`: Appended by `FillCloudySkySide`.
- `backEnd.skyRenderedThisView`: Set to `qtrue` in `RB_StageIteratorSky`; guards `RB_DrawSun` execution.

## Learning Notes

**1. Cube-Face Projection Primitives**
- `vec_to_st[6][3]` and `st_to_vec[6][3]` are **lookup tables** that encode the 3D↔2D mapping for each cube face. This is a pre-computed pattern match avoiding branches per vertex.
- Modern engines often use **cubemaps** (6-faced 3D textures) directly; here, each face is a separate 2D image, so projection math is necessary.

**2. Recursive Clipping as a Design Pattern**
- `ClipSkyPolygon(..., stage)` is a **six-stage pipeline** where each stage clips against one plane normal in `sky_clip[6]`. This is cleaner and more maintainable than a monolithic loop.
- At stage 6 (all planes processed), the fully-clipped polygon is accepted via `AddSkyPolygon`.
- The pattern generalizes: `if (stage == N_PLANES) { accept(); return; }` is a common idiom in graphics algorithms (e.g., frustum culling, clipping).

**3. Early-2000s OpenGL Era Techniques**
- **Immediate-mode drawing** (`qglBegin`/`qglEnd`) is deprecated in GL 3.0+ but was standard before VBOs (2003).
- **No LOD (level of detail)**: The sky is always rendered at full `SKY_SUBDIVISIONS=8` granularity; LOD selection is absent.
- **Fixed function pipeline**: All shading is done via OpenGL state machine, not shaders. Lighting is baked into textures.
- **Singular global `tess` buffer**: All geometry in a frame serializes through one interleaved vertex array; no streaming or GPU-resident buffers.

**4. Spherical Projection for Cloud Layers**
- `R_InitSkyTexCoords` intersects rays from the camera through each cloud grid point with a **virtual sphere** of radius 4096 (`SKY_SCALE`), computing where the ray hits the sphere.
- This creates the illusion that clouds are at a fixed distance (the sphere), avoiding visible "tiling" at the horizon.
- The math (normalized direction, sphere distance formula, `Q_acos` for angle) shows sophisticated use of vector algebra.

**5. Invariant: Camera-Centric Sky**
- Sky geometry **always** centers on `backEnd.viewParms.or.origin` and extends to `backEnd.viewParms.zFar / 1.75` (an arbitrary scale factor chosen to fit the virtual sphere comfortably).
- This ensures the player can never see the edge of the sky box or background geometry through seams.
- No parallax correction is necessary because the sky is infinitely far (from the player's perspective).

## Potential Issues

**1. Static Array Overallocation**
- `s_cloudTexCoords[6][9][9][2]` is hardcoded for `SKY_SUBDIVISIONS=8`, consuming ~3.5 KB per frame (6 faces × 81 vertices × 2 coords × 4 bytes).
- If `SKY_SUBDIVISIONS` changes, this array must be manually resized. No compile-time assertion guards against mismatch.

**2. Immediate-Mode Performance**
- `DrawSkySide` calls `qglBegin`/`qglEnd` per row of the mesh (8 calls per face in typical cases). Each pair incurs function-call overhead and state validation.
- Modern OpenGL (core profile) forbids this; a VAO + indexed draw call per face would be dramatically faster.

**3. Clipping Buffer Overflow**
- `ClipSkyPolygon` checks `if (nump > MAX_CLIP_VERTS-2)` and errors fatally.
- If a very large polygon straddles many clip planes, the vertex count can explode; `MAX_CLIP_VERTS=64` may be insufficient in pathological cases.
- The check happens only on entry; an intermediate stage could overflow if not caught.

**4. Floating-Point Precision in Projection**
- `MakeSkyVec` divides by `dv` without epsilon checks (e.g., `s = vecs[j-1] / dv` in `AddSkyPolygon`).
- Near-degenerate polygons (e.g., edges parallel to cube faces) could produce NaNs or infinities, corrupting bounds or geometry.

**5. Shared Static State Across Frames**
- `sky_mins`, `sky_maxs` are reset per-frame, but if `RB_ClipSkyPolygons` is called twice in one frame (e.g., for multiple sky surfaces in a complex shader), the bounds accumulate unpredictably.
- The code assumes **at most one sky surface per frame**, a fragile invariant.

**6. Missing Documentation of Cube Face Ordering**
- `sky_texorder[6] = {0,2,1,3,4,5}` reorders `outerbox[]` image indices, but the logic (why these indices, what each face represents) is undocumented.
- A future maintainer cannot easily add new cube face images or reorder them without understanding this magic number array.
