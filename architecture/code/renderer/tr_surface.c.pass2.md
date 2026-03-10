# code/renderer/tr_surface.c — Enhanced Analysis

## Architectural Role
This file is the **render back-end's surface tessellation dispatcher**—a critical junction where heterogeneous surface types (BSP faces, patch grids, MD3 meshes, dynamic sprites, beams, rails) converge into a single unified vertex/index buffer (`tess`) for downstream shader execution. It sits at the tail end of the renderer's front-end/back-end pipeline: after PVS frustum culling and sort-key ordering in `tr_main.c`/`tr_world.c`, but before `RB_EndSurface()` flushes the tess buffer into OpenGL commands. The file's core job is to translate each surface's abstract representation (compressed coordinates, patch control points, MD3 frames) into interleaved vertex attributes compatible with the fixed-function + multitexture pipeline of OpenGL 1.x.

## Key Cross-References

### Incoming (who depends on this file)
- **`tr_backend.c` (`RB_RenderDrawSurfList`/`RB_ExecuteRenderCommands`):** Main render-command executor calls `rb_surfaceTable[surfaceType](surface)` for every surface in the draw list after `RB_BeginSurface` has set up shader state.
- **`tr_cmds.c` (`RB_EndSurface`):** Flushes the tess buffer and immediately restarts a new one; called by `RB_CheckOverflow` when capacity is exceeded mid-surface.
- **`tr_animation.c` (`RB_SurfaceAnim`):** Wraps MD4 skeletal-animation deformation; called when `surfaceType == SF_MD4` (Team Arena).
- **`tr_image.c` (image load code):** Indirectly consumed by shader system, which references images bound in `DoRailCore`/`DoRailDiscs` (via `GL_Bind( tr.whiteImage )`).

### Outgoing (what this file depends on)
- **`tr_local.h` globals:**
  - `tess` (shaderCommands_t): Shared tessellation accumulator; written by all RB_Surface* functions.
  - `backEnd` (backEndState_t): Current entity state (`backEnd.currentEntity->e`, `backEnd.smpFrame`), view matrices (`backEnd.viewParms.or.axis`), mirror flag.
  - `tr` (trGlobals_t): Sin/cos lookup table (`tr.sinTable`), white placeholder image (`tr.whiteImage`).
- **Cvar lookups at draw time:** `r_railWidth`, `r_railSegmentLength`, `r_railCoreWidth`, `r_lodCurveError`.
- **Math utilities** (`q_math.c`): `VectorNormalizeFast`, `VectorArrayNormalize`, `PerpendicularVector`, `RotatePointAroundVector`, `MakeNormalVectors`.
- **OpenGL state:** `GL_Bind`, `GL_State`, immediate-mode `qglBegin/Vertex/Color/End` for procedural geometry (`RB_SurfaceBeam`, `RB_SurfaceAxis`).

## Design Patterns & Rationale

### 1. **Polymorphic Dispatch via Function Pointer Array**
`rb_surfaceTable[surfaceType]` enables the renderer to add new surface types without modifying core dispatch logic. This is classic OOP pattern in C—each surface type has a handler function (`RB_SurfaceTriangles`, `RB_SurfaceGrid`, `RB_SurfaceEntity`, etc.) registered in a vtable. **Tradeoff:** slightly higher branch unpredictability than a large switch, but excellent code locality per-surface-type and lower binary bloat.

### 2. **Double-Buffered Tess Buffer with Overflow-Driven Flush**
The `tess` accumulator is preallocated once to `SHADER_MAX_VERTEXES` capacity. Rather than dynamically growing, `RB_CheckOverflow` proactively flushes and restarts when the next surface won't fit. **Why:** Avoids allocation/deallocation per-surface and ensures predictable memory usage; SMP rendering becomes simpler (front-end can't resize a buffer the back-end is concurrently reading). **Consequence:** Very large surfaces (e.g., detailed Bézier grids) may internally cycle through multiple flush/restart pairs; see `RB_SurfaceGrid`.

### 3. **Conditional Normal Computation**
`RB_SurfaceTriangles` only copies normals into `tess.normal` if `tess.shader->needsNormal` is true. **Rationale:** Not all shaders require per-vertex normals (e.g., lightmapped diffuse); skipping the copy saves bandwidth and instruction cache footprint on normal-mapping-free passes. This is a low-level optimization exploiting shader-time knowledge during tessellation.

### 4. **Deferred Lerp + Quantization for MD3**
`LerpMeshVertexes` decompresses short-quantized xyz and latitude/longitude-encoded normals **at draw time**, then interpolates between `frame` and `oldframe` based on server-client time difference. **Why not precompute?** MD3 models are lightweight skinned assets; decompressing all frame data offline would balloons the file size. Doing it on-demand trades CPU cycles (cheap vector ops) for storage (tight memory budget on 2004 hardware). The Altivec path suggests this was a performance-critical inner loop on PowerPC (e.g., G4 Mac).

### 5. **Immediate-Mode GL for Procedural Entities**
`RB_SurfaceBeam` and `RB_SurfaceAxis` (not shown but referenced) **bypass the tess buffer entirely** and issue direct `qglBegin/glVertex/glEnd` sequences. **Rationale:** These are rare, non-batched entities (debug axes, transient beams) that don't benefit from interleaving; immediate mode is simpler for one-off geometry. By contrast, rails and lightning are tessellated into `tess` because they can batch with other translucent surfaces.

### 6. **LOD via Precomputed Error Metrics**
`RB_SurfaceGrid` calls `LodErrorForVolume` to decide subdivision level. The grid mesh (`srfGridMesh_t`) ships precomputed error bounds for all sub-patches; the renderer simply queries them at draw time. **Design rationale:** LOD decision at CPU time (frame boundary) vs. on GPU allows coarse/fine grids to coexist in the same batch without per-pixel branching.

## Data Flow Through This File

**Input streams:**
1. **Sorted draw list** (from `tr_main.c`): Contains pointers to all visible surfaces and the shader to apply.
2. **`backEnd.currentEntity`:** The entity context for the current surface (origin, rotation, shader RGBA, animation frame).
3. **View parameters** (`backEnd.viewParms`): Camera orientation, frustum, mirror state for billboard orientation.

**Transformation/accumulation:**
- Each surface type's handler parses its abstract representation and writes interleaved vertex data (XYZ, normal, ST, lightmap ST, color) into `tess.xyz`, `tess.normal`, `tess.texCoords`, `tess.vertexColors`.
- Index data is written to `tess.indexes` to form triangles (fans for grids/polys, strips for quad sequences).
- Special flags like `tess.dlightBits` accumulate per-vertex dynamic light influence.

**Output stream:**
- When `tess` reaches capacity (or at end-of-scene), `RB_EndSurface()` is called (not in this file, but assumed to be the next step).
- `RB_EndSurface` invokes the shader's compiled multi-pass command list, which reads `tess` vertex data and issues OpenGL state + draw calls.

## Learning Notes

### Idiomatic Patterns (Early 2000s OpenGL)
- **Fixed-function pipeline assumptions:** Every vertex has XYZ, normal, two sets of ST (base texture, lightmap), and RGBA color. Modern engines (post-shader-era) decouple attributes per material.
- **Immediate-mode GL fallback:** `qglBegin/End` is used for rare, non-batched primitives. By 2005, this was already discouraged in favor of VBOs, but Q3A predates that era.
- **Vertex format packing:** Normals are stored as 3 floats (expensive), not quantized shorts. Lightmap coords are separate, not baked into vertex color.

### Differences from Modern Engines
| Aspect | Q3A (tr_surface.c) | Modern (e.g., Unreal/Unity) |
|--------|-------------------|----------------------------|
| **Surface dispatch** | Runtime function-pointer vtable (`rb_surfaceTable`) | Compile-time template specialization or data-driven material pipelines |
| **Buffer management** | Single global `tess` buffer, flush-on-overflow | Multi-buffering, persistent mapped GPU buffers, compute shaders for transformation |
| **Mesh deformation** | CPU-side MD3 frame lerp in `LerpMeshVertexes` | GPU vertex shaders with per-instance transforms and time uniforms |
| **LOD** | Precomputed error metrics queried at CPU time | Compute-shader-driven mesh clustering or hardware tessellation shaders |
| **Procedural geometry** | Immediate-mode GL (beams) or hand-rolled tess (rails) | Procedural mesh generators or GPU-driven rendering via indirect dispatch |
| **Normal handling** | Optional per-shader, copied only if needed | Always present; packed into vertex stream or computed in shader |

### Connections to Engine Concepts
- **Scene graphs / Culling:** This file assumes input is a **sorted draw list** pre-filtered by PVS and frustum. The scene graph is implicit: BSP tree in `tr_world.c` is traversed front-to-back.
- **Instancing:** MD3 models are **not instanced**; each entity instance generates its own tessellated geometry. No transform batching.
- **ECS:** No entity component system here; instead, `backEnd.currentEntity` is mutable global state passed frame-to-frame. Contrast: modern engines store component data in dense arrays.

## Potential Issues

### Subtle Issues (Inferable from Code + Context)

1. **Uninitialized Tess Pointer Arithmetic** (tr_surface.c:318–360, `LerpMeshVertexes`)
   - Pointers like `xyz`, `normal`, `texCoords`, `color` are advanced by 4 floats per vertex in a loop.
   - If `tess.numVertexes` is not properly aligned to the layout, pointer arithmetic could read/write adjacent fields.
   - **Mitigation:** Assumes all `tess` arrays are preallocated together and indexing is consistent; relies on caller to ensure layout invariants.

2. **Conditional Normal Lerp Without Re-normalization** (tr_surface.c, `LerpMeshVertexes`)
   - When `backlerp != 0`, normals are linearly interpolated in the scalar path (Altivec path calls `VectorArrayNormalize` after).
   - **Issue:** Linearly interpolated unit vectors are not unit vectors; the scalar path has a commented-out `VectorNormalize`.
   - **Impact:** Lighting artifacts on frames between keyframes (though subtle; the shader may mask this).
   - **Root cause:** Performance optimization to avoid per-vertex normalization; Altivec path normalizes in bulk.

3. **Missing Overflow Check in `DoRailDiscs`** (tr_surface.c ~line 340)
   - The loop `for (i = 0; i < numSegs; i++)` calls `RB_CHECKOVERFLOW(4, 6)` but doesn't account for prior loop iterations.
   - **Issue:** If the first iteration passes the check but later iterations accumulate beyond `SHADER_MAX_VERTEXES`, the function could overrun the buffer.
   - **Actual risk:** `numSegs` is derived from `r_railSegmentLength->value` and map size; in practice, rarely exceeds a few hundred. Overflow is unlikely but theoretically possible.
   - **Note:** This is a minor issue; the first-pass analysis doesn't flag it, so it may not manifest in real gameplay.

4. **Shared Global State Assumes Single-Threaded Access**
   - `backEnd.currentEntity` is a mutable global; all RB_Surface* functions read it without locking.
   - **In SMP mode:** The front-end and back-end run on separate threads. The front-end (tr_main.c) iterates the draw list and fills `tess` while the back-end (tr_backend.c) drains it.
   - **Invariant:** By the time the back-end reads `tess`, no further modifications are made. This is guaranteed by frame synchronization barriers in `GLimp_FrontEndSleep/WakeRenderer`.

5. **Litmus Test for MD3 Quantization Precision**
   - MD3 verts are stored as signed shorts (16-bit). After dequantization and interpolation, precision loss may accumulate over many frames.
   - **In practice:** Not an issue for characters (models update every frame); only noticeable for static MD3s or very long demos.

---

## Summary: Role in Broader Renderer
This file is a **funnel:** it takes heterogeneous surface representations (BSP, curves, models, sprites, procedural) and converts them into a unified tessellation format for a fixed shader execution pipeline. Its success hinges on efficient overflow management, correct vertex attribute routing, and tight integration with the upstream sort/cull pass (`tr_main.c`). It is NOT a scene-graph manager, frustum culler, or shader compiler—those live elsewhere. Its isolation of tessellation logic makes the renderer modular and permits per-surface-type optimizations (e.g., skipping normals, conditional lerp) that benefit from shader-time knowledge.
