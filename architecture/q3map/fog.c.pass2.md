# q3map/fog.c — Enhanced Analysis

## Architectural Role

This file implements **offline fog volume geometry processing** within the `q3map` BSP compiler. It clips all drawn surfaces (patches and faces) against fog brushes during the map compilation pipeline, determining which geometry is visibly fogged at runtime. The module bridges the gap between the level design's fog volumes and the runtime engine's fog rendering by pre-processing spatial relationships before the final BSP output is written.

## Key Cross-References

### Incoming (who depends on this file)
- **`q3map/bsp.c`** calls `FogDrawSurfs()` in the post-surface-pruning, pre-tjunction phase
- **Map compilation pipeline** invokes `FogDrawSurfs()` as part of the ordered BSP processing sequence

### Outgoing (what this file depends on)
- **`qbsp.h`** (tool header) for struct definitions: `mapDrawSurface_t`, `bspbrush_t`, `drawVert_t`, `plane_t`, `shaderInfo_t`, `winding_t`
- **Other q3map modules**:
  - `DrawSurfaceForMesh()`, `DrawSurfaceForSide()` — surface factory functions (likely in another q3map module)
  - `FreeMesh()`, `TransposeMesh()`, `InvertMesh()` — mesh utility functions
  - `ClipWindingEpsilon()`, `FreeWinding()`, `AllocWinding()` — geometric clipping primitives
  - `ComputeAxisBase()` — texture matrix computation
- **Global state**:
  - `mapDrawSurfs[]`, `numMapDrawSurfs` — the master draw-surface array being processed
  - `mapplanes[]` — plane data from BSP
  - `entities[]` — entity array containing all brushes
  - `dfogs[]`, `numFogs` — output fog array being populated
  - Counter globals: `c_fogFragment`, `c_fogPatchFragments`

## Design Patterns & Rationale

**Recursive Geometric Clipping**: The code uses a two-phase approach:
1. **Patch clipping** (`ChopPatchByBrush`): Splits Bézier patches by finding crossing grid lines and recursively subdividing
2. **Face clipping** (`ChopFaceByBrush`): Uses winding-based half-space clipping to determine interior/exterior fragments

**Why this design?** Patches and polygonal faces require different clipping strategies due to their geometric representation. Patches clip along grid lines to preserve patch structure; faces use arbitrary plane clipping. Both accumulate "outside" fragments as new draw surfaces and retain only the portion inside the fog brush.

**Texture coordinate preservation**: `ChopFaceByBrush` re-computes texture coordinates for clipped faces using either:
- Old-style brush vecs (`s->vecs[0..1]` + `s->vecs[i][3]` offset)
- Brush primitives texture matrix (`s->texMat`)

This ensures clipped geometry remains properly textured relative to the original brush.

## Data Flow Through This File

**Input → Transform → Output:**

1. **Input**: All `mapDrawSurfs[]` accumulated from prior BSP phases; all `entities[0].brushes` containing `CONTENTS_FOG` flags
2. **Processing per fog brush**:
   - Extract brush planes and shader info
   - For each existing draw surface: clip against the brush's top/bottom planes (patches) or all planes (faces)
   - Accumulate "outside" fragments as new draw surfaces linked to the same side
   - Replace the original surface with only the clipped interior portion
3. **Output**: 
   - Modified `mapDrawSurfs[]` array (in-place mutation)
   - New draw surfaces appended for all exterior fragments
   - Updated fog array `dfogs[]` mapping brushes to shaders
   - Counters `c_fogFragment`, `c_fogPatchFragments` for diagnostics

## Learning Notes

**Idiomatic to this era and engine:**
- **Compile-time geometry decomposition**: Q3 pre-processes all spatial relationships offline; the runtime engine reads pre-computed, pre-clipped geometry. Modern engines often use dynamic fog volumes with runtime clipping.
- **Dual representation handling**: Patches (smooth grids) and faces (polygons) coexist as first-class draw surface types, each with specialized clipping logic.
- **Conservative brush-centric clipping**: Fog only clips against the top/bottom (5th/6th) brush planes to avoid "messy patch clipping issues" (see comment at line 237). This is a deliberate simplification accepting some over-fogging in exchange for stability.
- **Winding-based polygon clipping**: The `winding_t` and `ClipWindingEpsilon` approach (epsilon-based halfspace splitting) is a common technique in 1990s–2000s game engines for robust polygon clipping.

**Modern contrast**: Contemporary engines would likely:
- Use GPU-based fog rendering with distance/height fog shaders rather than pre-clipped geometry
- Apply fog at render time using volume primitives rather than geometry modification

## Potential Issues

1. **Limited patch clipping robustness**: Only clips by top/bottom planes, potentially leaving patches partially inside fog boundaries. The comment "to avoid some messy patch clipping issues" suggests this was a pragmatic trade-off.
2. **Texture coordinate adjustment**: The `mins[]` floor adjustment assumes `si->globalTexture` is meaningful; if shader info is malformed, clamping could produce visual artifacts.
3. **Memory accumulation**: Clipped fragments are allocated but never explicitly freed in `FogDrawSurfs()` loop—they rely on implicit deallocation at tool shutdown via Hunk/zone allocators.
