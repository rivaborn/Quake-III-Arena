# code/renderer/tr_mesh.c — Enhanced Analysis

## Architectural Role
This file is the **front-end MD3 rendering gateway** in Quake III's two-phase renderer. It bridges the generic scene traversal system (`tr_scene.c`/`tr_main.c`) to the deferred draw-surface submission pipeline, implementing entity-level culling, animation frame validation, shader resolution, and LOD selection for skinned triangle mesh models. No GPU commands are issued here—only `drawSurf_t` entries are enqueued to `tr.pc.drawSurfs` for later sorting and back-end execution.

## Key Cross-References
### Incoming (who depends on this file)
- **`tr_scene.c` / `tr_main.c`**: Scene traversal loop calls `R_AddMD3Surfaces` once per MD3 entity after setting `tr.currentModel` pointer. This is part of the leaf-walking frustum cull in the BSP traversal phase.
- **`tr_model.c`**: Entity model loading/binding; sets `tr.currentModel->md3[lod]` array prior to scene submission.

### Outgoing (what this file depends on)
- **`R_CullLocalPointAndRadius`, `R_CullLocalBox`** (`tr_main.c`): Frustum cull tests for bounding sphere and AABB.
- **`R_AddDrawSurf`** (`tr_main.c`): Enqueues sorted draw surface; called 1–3 times per surface (shadow passes + main).
- **`R_SetupEntityLighting`** (`tr_light.c`): Computes ambient/directional light grid sample for dynamic light application in back-end.
- **`R_GetShaderByHandle`, `R_GetSkinByHandle`** (`tr_shader.c`/`tr_image.c`): Shader and skin asset lookup.
- **`RadiusFromBounds`** (`q_math.c`): Sphere radius from AABB; used in LOD projection.
- **`ProjectRadius`** (local): Transforms world-space sphere radius through projection matrix to normalized screen coverage.
- **`tr.viewParms`, `tr.currentModel`, `tr.shaders`, `tr.world`**: Global renderer state (view, model LOD/shader arrays, fog world).

## Design Patterns & Rationale

**Hierarchical Culling**: Sphere-then-AABB-then-per-surface ordering reduces fill-rate waste. Sphere test is skipped for non-uniform transforms (`nonNormalizedAxes`), reflecting precision/robustness concerns.

**Lazy LOD Computation**: LOD index computed per-entity, not per-surface. The formula `flod = 1.0 - projectedRadius * lodscale; lod = clamp(flod * numLods)` biases toward detail when object is large on screen; clamping ensures 0-based indexing. Objects intersecting the near plane default to lowest-quality LOD (`flod=0`), an ad-hoc solution for weapon models.

**Deferred Shader Resolution**: Rather than precomputing shaders during model load, resolution is per-frame-per-surface, allowing dynamic customization (`RF_CUSTOM_SHADER`, custom skins). Fallback hierarchy: custom shader → custom skin lookup → embedded MD3 shader → default shader.

**Shadow Submission Decoupling**: Shadow surfaces (stencil mode 2, projection mode 3) are submitted independently of the main surface, allowing shadow rendering with different frustum/depth rules. Personal model restriction (`!personalModel`) applies only to main surface, not shadows (except stencil mode, which explicitly checks `!personalModel`).

**Frame Validation as Safety Net**: Rather than asserting on out-of-range frames, the code silently clamps them to [0, 0], a defensive choice reflecting netcode robustness concerns (e.g., client/server desync recovery).

## Data Flow Through This File

**Input**: `trRefEntity_t *ent` with:
- `ent->e.frame`, `ent->e.oldframe`: Frame indices (may be invalid or wrapped).
- `ent->e.origin`: World position for LOD projection and fog membership.
- `ent->e.renderfx`: Flags (RF_THIRD_PERSON, RF_WRAP_FRAMES, RF_NOSHADOW, RF_DEPTHHACK, RF_SHADOW_PLANE).
- `ent->e.customShader`, `ent->e.customSkin`, `ent->e.skinNum`: Shader override tokens.
- `tr.currentModel`: Pre-selected model (MD3 header + LOD array).

**Transformation**:
1. Frame validation: clamp/wrap to [0, numFrames-1]; emit developer warning if invalid.
2. LOD selection: compute projected radius, apply bias, select LOD from `tr.currentModel->md3[lod]`.
3. Culling: test bounding sphere (or skip if scaled) and AABB; return early if fully outside frustum.
4. Lighting setup: invoke per-entity light grid sampling (unless personal model and shadows disabled).
5. Per-surface iteration:
   - Resolve shader (custom > custom skin > embedded > default).
   - Emit shadow draw surfaces (if opaque, no fog, no culling flags).
   - Emit main draw surface (unless personal model).

**Output**: Multiple `drawSurf_t` entries pushed to renderer sort queue via `R_AddDrawSurf`, each tagged with:
- Surface geometry pointer (for back-end vertex/index unpacking).
- Shader reference (for state sorting).
- Fog index (0 = no fog).
- Personal model flag (for SMP synchronization).

## Learning Notes

**Q3A-Era Idiosyncrasy**: The MD3 format stores per-frame bounding data (`md3Frame_t.localOrigin`, `md3Frame_t.radius`, `md3Frame_t.bounds[2]`), allowing frame-precise culling. Modern engines typically use a single world-space AABB per model. This per-frame approach reflects Q3A's emphasis on skeletal animation and dynamic hitbox precision.

**LOD as Screen-Space Coverage**: The `ProjectRadius` function manually applies the projection matrix to estimate screen coverage, a technique older than GPU-driven LOD selection. Scaling by `r_lodscale` cvar allows runtime tuning; clamping to [1.0, 20.0] suggests empirical balancing.

**Fog Membership as Linear Search**: `R_ComputeFogNum` iterates world fog list (typically 5–10 fogs) with a simplistic AABB overlap test. This predates spatial hashing; acceptable for typical map fog counts but O(n) per entity.

**Dual-Frame Culling Heuristic**: When `oldframe != newframe`, both spheres are tested independently; only if both agree on full culling (`CULL_OUT`) is the entity rejected. This guards against animation pop-in during frame transitions—a conservative strategy reflecting frame-rate independence concerns.

## Potential Issues

1. **Asymmetric Shadow/Main Submission**: Shadow surfaces check `!personalModel` only for stencil mode (2), not projection mode (3). A personal model (e.g., view weapon) with `r_shadows=3` and `RF_SHADOW_PLANE` set could still cast a shadow on the world—likely unintended.

2. **Non-Normalized Axis FIXME**: `R_ComputeFogNum` skips per-frame bounds adjustment for non-uniformly-scaled entities (see comment). Scaled models may occupy the wrong fog volume.

3. **Frame Wrapping vs. Modulo**: `RF_WRAP_FRAMES` uses `%` modulo rather than bit-wise AND, incurring a division per frame if set. Not a performance blocker, but suggests defensive coding over optimization.

4. **Shader Validation Noise**: Mismatched skin shader lookups emit `PRINT_DEVELOPER` warnings repeatedly each frame if a custom skin doesn't contain all surfaces. Spammy on maps with custom-skinned entities; no throttling.

5. **Shadow Fog Exclusion**: Shadows are never rendered into fog (`fogNum == 0` check). Underwater or foggy enclosed areas lose shadow information—a visual simplification with clear tradeoffs but undocumented.
