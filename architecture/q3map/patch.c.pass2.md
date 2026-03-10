# q3map/patch.c — Enhanced Analysis

## Architectural Role

This file implements the **patch parsing and grouping subsystem** within the offline BSP compiler. It bridges the tokenized `.map` file format (parsed by `q3map/map.c`) and the intermediate patch representation fed into the renderer's Bézier surface pipeline. Patches parsed here become `mapDrawSurface_t` entries that are eventually serialized into the `.bsp` file; the grouping logic ensures edge-connected patches coordinate their tessellation LOD to prevent visual seams at runtime.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/map.c**: The main entity/map parser calls `ParsePatch()` when encountering a `patchDef2` or `patchDef3` entity surface definition.
- **q3map's main loop**: Invokes `PatchMapDrawSurfs()` for each entity after all brushes and patches are parsed, to perform grouping before BSP finalization.
- **Offline compilation only**: No runtime code calls these functions; the entire module is compile-time-only.

### Outgoing (what this file depends on)
- **Tokenization layer** (`MatchToken`, `GetToken`, `Parse1DMatrix`, `UnGetToken`): Implemented elsewhere in q3map's parsing infrastructure.
- **Mesh utilities** (`CopyMesh`, `FreeMesh`, `PutMeshOnCurve`, `MakeMeshNormals`): Low-level mesh manipulation; likely in `q3map/mesh.c`.
- **Surface allocation** (`AllocDrawSurf`): Factory for `mapDrawSurface_t`; driver-level.
- **Shader lookup** (`ShaderInfoForShader`): Looks up material definitions by name; integrates with the shader system.
- **Global state**:
  - `mapent`: Current entity context (pointer to `entity_t`).
  - `mapIndexedShaders[]` / `numMapIndexedShaders`: Deduplication table for shader names per-map.
  - `noCurvoBrushes`: Cvar controlling whether curve brushes are compiled at all.
  - `g_bBrushPrimit`: Enum controlling map format variant (old brushes vs. brush primitives).

## Design Patterns & Rationale

1. **Lazy Normal Computation** (lines 47–60): `DrawSurfaceForMesh` copies the mesh, projects verts onto the Bézier surface with `PutMeshOnCurve`, and recomputes normals from the corrected positions before writing them back to the original. This defers expensive normal recalculation until the final surface is generated, avoiding the cost for intermediate operations.

2. **Stateless Grouping via Adjacency Matrix** (lines 182–234): The `bordering` matrix encodes patch-to-patch connectivity as a dense 2D boolean array. Contrast with runtime LOD systems (like modern engines) that recompute adjacency on-demand; here, offline compilation amortizes the O(n²) connectivity pass into a one-time precomputation. Flood-filling groups via `GrowGroup_r` ensures all transitively connected patches form a single LOD unit.

3. **Map Format Abstraction** (lines 131–137): Conditional parsing of epairs based on `g_bBrushPrimit` allows the tool to consume both legacy Q3A `.map` files (pre-brush-primitives) and newer Team Arena variants without branching the core parser logic.

4. **Shader Caching** (lines 127–128): All patch shaders are cached in `mapIndexedShaders` for deduplication across the map. This table is likely fed into the BSP header's shader directory.

## Data Flow Through This File

1. **Parsing** (`ParsePatch`):
   - Tokenizer feeds patch definition: shader name, 5-element control grid size/format, vertex grid.
   - Vertices allocated and filled in column-major order (outer loop `j` for width, inner `i` for height).
   - Shader name added to global dedup table; patch wrapped in `parseMesh_t` and linked to the current entity.
   - Returns without creating draw surfaces yet (deferred to grouping phase).

2. **Grouping** (`PatchMapDrawSurfs`):
   - Collects all patches from a single entity into a `meshes[]` array.
   - Builds `bordering` adjacency matrix by comparing vertex positions (±1.0 tolerance) between all patch pairs.
   - Recursively floods groups: each unvisited patch seed calls `GrowGroup_r` to mark all reachable neighbors.
   - For each group, bounds the vertices, then calls `DrawSurfaceForMesh` to produce the final `mapDrawSurface_t` and link it into the BSP surface list.

3. **Output**:
   - `mapDrawSurface_t` entries are accumulated in a global list and later serialized into the BSP as draw surfaces.
   - Groups ensure the renderer's LOD system sees all edge-adjacent patches as a coherent unit.

## Learning Notes

- **Pre-runtime tessellation is absent**: Unlike modern engines (which tessellate patches at render-time), Q3A bakes static tessellated grids into the BSP. The "patch" stored in the BSP is a fixed `(patchWidth × patchHeight)` grid of vertices, not a parametric surface. This trades flexibility for predictable memory and performance offline.

- **Idiomatic offline-tool patterns**: The stateless traversal (tokenizer → parser → grouper → serializer) reflects how early-2000s tools were written—no caching, no dependency graphs, pure pass-oriented data flow. The global state (`mapent`, `noCurveBrushes`) would be encapsulated into a context struct in modern C.

- **LOD grouping rationale**: In the renderer (`code/renderer/tr_curve.c`), patches in the same group share tessellation decisions to avoid T-junction cracks. At runtime, the grouping metadata is baked into the BSP and used by `RE_BeginSurfaces` to ensure a group's patches are tessellated consistently.

- **Connection via Bézier mesh, not topology**: Patches are linked not by explicit edge descriptors, but by spatial vertex proximity. This works because valid Q3A maps respect vertex snapping, but it's fragile—floating-point errors can break intended adjacencies.

## Potential Issues

- **Float comparison fragility** (line 224): The 1.0-unit threshold for vertex matching is hand-tuned and may fail if vertices are slightly offset due to map editor rounding or coordinate transformation. No epsilon/tolerance parameterization.

- **O(n²) vertex comparison** (lines 217–233): For each of `patchCount²` patch pairs, every vertex of one is compared against every vertex of the other. With 100+ patches, this could be slow; a spatial hash would scale better. However, this runs once offline, so the impact is minimal in practice.

- **Memory leak in brush-primitive path** (line 138 comment): The code acknowledges that `ParseEpair()` allocates memory that is never freed if brush-primitives mode is active. This is a known limitation of the offline tool, not a critical bug (the tool exits after compilation).
