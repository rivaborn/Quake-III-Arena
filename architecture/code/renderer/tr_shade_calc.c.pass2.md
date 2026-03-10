# code/renderer/tr_shade_calc.c — Enhanced Analysis

## Architectural Role

This file is the **shader evaluation back-end**, executing the per-vertex procedural pipeline for all deformations, color/alpha generation, and texture coordinate synthesis. It forms the critical bridge between the shader AST (`shader_t` defining stages) and the tessellated geometry ready for submission to OpenGL. All functions operate on the global `tess` buffer, which accumulates vertex data from the front-end surface traversal phase (BSP faces, meshes, sprites, etc.). The file is called during `tr_shade.c`'s per-surface loop, after geometry is assembled but before the back-end command queue is built.

## Key Cross-References

### Incoming (who depends on this file)
- **`tr_shade.c`** — primary caller; invokes `RB_DeformTessGeometry()` once per surface, then calls individual `RB_Calc*Color`, `RB_Calc*Alpha`, `RB_Calc*TexCoords` functions during shader stage iteration
- **`tr_backend.c`** — indirectly; the back-end command buffer is populated with geometry modified by this file's functions
- **Renderer initialization (`tr_init.c`)** — precomputes waveform lookup tables (`tr.sinTable`, etc.) that this file reads; these tables are essential performance optimization

### Outgoing (what this file depends on)
- **`tr_local.h`** — core types, globals `tess` (shaderCommands_t), `backEnd`, `tr`, `ri`; also `WAVEVALUE` macro definition
- **`tr_shader.c`** — shader definition parsed into `shader_t` stages; this file reads `tess.shader->deforms[]` and stage parameters
- **`tr_noise.c`** — `R_NoiseGet4f()` for Perlin-style procedural deformation in `RB_CalcDeformNormals`
- **`tr_shade.c`** — `RB_CalcTransformTexCoords()` helper (matrix multiply for texture coordinate warping)
- **Other renderer functions** — `RB_AddQuadStamp`, `RB_AddQuadStampExt` (sprite billboarding), `RB_ProjectionShadowDeform` (shadow volume generation)
- **Math library** — `myftol()` (float→int conversion, often x86-specific), `Q_rsqrt()` (fast 1/√x), vector operations (`VectorScale`, `DotProduct`, `CrossProduct`, `VectorNormalize`)

## Design Patterns & Rationale

### 1. **Lookup Table Waveforms (Performance via Precomputation)**
All periodic waveforms (sin, triangle, square, sawtooth) use precomputed tables indexed via `WAVEVALUE` macro. This avoids expensive trigonometric calls per vertex. The macro combines phase offset, shader time, and frequency into a table index in a single inline expression—extremely cache-friendly and common in Q3-era engines that were CPU-bound before GPU shaders became standard.

```c
#define WAVEVALUE( table, base, amplitude, phase, freq ) \
  ((base) + table[ myftol( (( (phase) + tess.shaderTime * (freq) ) * FUNCTABLE_SIZE ) ) & FUNCTABLE_MASK ] * (amplitude))
```

### 2. **Two-Path Branching in Deformations**
Functions like `RB_CalcDeformVertexes` branch on `frequency == 0`:
- **Frequency 0**: Simple wave evaluated once, scales all normals uniformly (fast path)
- **Frequency > 0**: Per-vertex phase offset based on position, creating ripple/wave effects (slower but more interesting)

This conditional avoids the per-vertex `off = xyz[0] + xyz[1] + xyz[2]` calculation for static deforms.

### 3. **Stateful Global Mutation**
All functions are **void** and mutate `tess` directly. No return values, no functional composition. This is idiomatic to Q3's data-oriented back-end design where the tessellator is a mutable staging buffer.

### 4. **Sprite Billboarding via Geometry Reconstruction**
`AutospriteDeform` and `Autosprite2Deform` completely rebuild quad geometry:
- Flush original vertex/index counts to zero
- Iterate original quads (4 verts at a time)
- Recompute world-space quad orientation (camera-facing for Autosprite, long-axis-pivoting for Autosprite2)
- Call `RB_AddQuadStamp` to emit new geometry

This avoids per-vertex shader overhead—geometry is deformed on the CPU before submission.

### 5. **Entity-Relative Coordinate Transforms**
Functions like `GlobalVectorToLocal` transform view vectors into the current entity's local space when needed (non-world entities). This allows billboarding to work correctly for entities with arbitrary rotation matrices.

## Data Flow Through This File

**Input:**
- `tess.xyz`, `tess.normal`, `tess.texCoords[0..15]`, `tess.vertexColors` — accumulated vertex data from surface tessellation
- `tess.shader` — parsed shader definition with deform stages and color/alpha/texcoord generators
- `backEnd.refdef.time` (shader time, synchronized across frame)
- `backEnd.or`, `backEnd.currentEntity` — entity transformation matrices

**Processing (sequential stages):**
1. **Deformations** (`RB_DeformTessGeometry` dispatcher):
   - Deform vertex positions and normals based on stage type (wave, bulge, move, autosprite, text, shadow)
   - May rebuild index/vertex counts entirely (autosprite) or mutate in-place
2. **Color Generation** (per-stage RGB):
   - Entity color blend → ambient/directional light multiplication → waveform modulation
   - Stores result in `tess.vertexColors[i]` as bytes [R, G, B, A]
3. **Alpha Generation** (per-stage A channel):
   - Entity alpha → specular lighting calculation or waveform evaluation
   - Stores in `tess.vertexColors[i][3]`
4. **Texture Coordinate Generation** (per-stage UVs):
   - Environment mapping, fog coordinates, turbulence, scrolling, rotation, scaling, stretching
   - Transforms `tess.texCoords[stageNum]` in-place or into stage-specific arrays

**Output:**
- Modified `tess` ready for back-end command submission
- OpenGL draw call receives the transformed vertices, normals, colors, and UVs
- No intermediate data structures; direct in-place modification

## Learning Notes

### Idiomatic to Quake III / Early-2000s Game Engines
- **No GPU shaders**: All per-vertex procedural work happens on CPU. Modern engines move this to fragment/vertex shaders.
- **Lookup tables over functions**: Trigonometry was expensive; tables cached the result. Trade memory for speed.
- **Hardcoded light origin**: The global `lightOrigin` for specular (marked FIXME) is a hack—proper engines compute dynamic light directions per-frame or per-light.
- **Geometry reconstruction for effects**: Billboarding, text, and sprites are rebuilt every frame on CPU, not instanced/parameterized at draw time.
- **SMP-aware code paths**: Some functions have AltiVec (PowerPC SIMD) optimizations (`RB_CalcDiffuseColor`) for multi-platform shipping.

### Modern Equivalent Patterns
- **Lookup tables** → vertex/fragment shaders with texture samplers or trigonometric instructions
- **Per-vertex deformation** → geometry/displacement shaders or compute shaders
- **Sprite billboarding** → shader-side vertex manipulation or geometry instancing
- **Entity-relative transforms** → model-view matrix in shader

### Key Algorithmic Insights
- **`WAVEVALUE` macro is the hot path**: Evaluated millions of times per frame; every instruction matters. The `&FUNCTABLE_MASK` bit-mask avoids modulo, and `myftol` is an x86-optimized int-conversion.
- **Normal-based deformation is directional**: Bulge and wave deformations always scale along the vertex normal, enabling effects like water ripples or breathing.
- **Spritesheet text rendering** (`DeformText`): Uses a 16×16 character grid (0–255 ASCII, 4-bit row/col) to map a single quad into multiple characters. Clever but limited to bitmap fonts.
- **Fog texture coordinates**: Computed as (eye distance, fog volume depth) for a specialized 1D or 2D fog lookup texture—avoids per-pixel fog calculation.

## Potential Issues

1. **Hardcoded `lightOrigin` Global** (lines ~1050–1100, `RB_CalcSpecularAlpha`):
   - Marked with FIXME; specular lighting is computed from a fixed world-space point, not the actual dynamic light direction. This breaks realism for moving lights.
   - Workaround: Engine likely uses separate dynamic light passes in shaders.

2. **Unvalidated Sprite Vertex Counts**:
   - `AutospriteDeform` warns on odd vertex counts but doesn't error; may produce malformed quads if shader author provides wrong-sized input.

3. **No SIMD for Most Vertex Operations**:
   - Only `RB_CalcDiffuseColor` has AltiVec path; other functions iterate scalar. Modern SSE2+ could parallelize 4 vertices per iteration.

4. **Text Rendering Assumes 4-Vertex Quad Input**:
   - `DeformText` iterates quad-by-quad; if a shader deform stage precedes text deformation, geometry shape assumptions break.

---

## Summary

`tr_shade_calc.c` is the procedural **vertex transformation engine** for Quake III's shader system. Its lookup-table-driven waveforms, per-vertex deformations, and colorization pipeline are quintessentially early-2000s real-time rendering—optimized for CPU-bound geometry processing before GPU shaders became universal. The file demonstrates idiomatic patterns (table-based math, stateful global buffers, geometry reconstruction for effects) that heavily influenced subsequent game engines, though modern systems would push most of this work to GPU shaders and instancing.
