# q3map/terrain.c — Enhanced Analysis

## Architectural Role

This terrain compiler module is a specialized offline-tool component (part of `q3map/`, the BSP compiler) that transforms raw terrain entity definitions into optimized surface meshes for the renderer. It bridges the map editing layer (entity definitions + alpha maps) to the BSP emission pipeline (`mapDrawSurface_t` structures), handling layer blending, vertex deduplication, triangle optimization, and coordinate space transformations. Unlike the runtime engine's renderer, this file has no performance constraints—it prioritizes correctness and mesh quality for offline computation.

## Key Cross-References

### Incoming (what calls terrain.c functions)
- `q3map/` build system calls the main terrain entry point(s) during BSP compilation
- Likely invoked from `q3map/bsp.c` or `q3map/writebsp.c` during the surface emission phase (after BSP tree construction but before final output)
- The file operates on global `mapent` (current entity being compiled), `mapplanes[]` (BSP plane array), and `surfaces[]` (local vertex/index buffers)

### Outgoing (what terrain.c depends on)
- **Image loading:** `Load32BitImage()`, `Load256Image()`, `ExtractFileExtension()` — from common or `q3map/` utilities, for alpha map asset loading
- **Shader system:** `ShaderInfoForShader()` — resolves shader definitions for layered texture naming
- **Texture coordinates:** `QuakeTextureVecs()` — computes planar UV vectors (must match runtime renderer's formula)
- **Surface allocation:** `AllocDrawSurf()` — places final mesh into the global BSP surface list
- **Collision/BSP data:** `FindFloatPlane()`, global `mapplanes[]` — for normal vectors and plane lookups
- **Math utilities:** from `q_shared.c` / `q_math.c` — `VectorAdd`, `CrossProduct`, `VectorNormalize`, floor operations

## Design Patterns & Rationale

**Triangle optimization pipeline:** The code attempts three triangle strip strategies (degenerate test → tristrip order → tri-fan fallback), reflecting the offline-compilation philosophy: *correctness over speed*. `IsTriangleDegenerate()` uses a cheap cross-product area threshold (10 units²) to reject collinear/backwards faces that would cause Z-fighting or cracks in the renderer. This is a defensive strategy unlikely in final assets but essential for robustness.

**Vertex deduplication:** `CompareVert()` with floating-point tolerance (`floor(v + 0.1)`) avoids duplicate vertices across multiple terrain surfaces sharing edges. This pattern mirrors runtime mesh optimization but happens offline.

**Texture coordinate duality:** `EmitTerrainVerts()` accepts a `projecttexture` boolean to choose between two UV schemes:
- **Projected (XY plane):** Uses a uniform quadrilateral grid for all terrain (via `QuakeTextureVecs(xynorm, ...)`), decoupling texture density from surface orientation
- **Face-normal:** Uses pre-computed `side->vecs[]` texture vectors, preserving per-surface parametrization

This dual mode accommodates both heightfield terrain and arbitrary brush-based terrain geometry.

## Data Flow Through This File

1. **Input:** Entity definition (`mapent`) with `"alphamap"` key (image path), `"layers"` count, optional `"min"`/`"max"` overrides
2. **Load phase:** `LoadAlphaMap()` reads TGA (32-bit RGBA) or 8-bit palette images, quantizes to layer indices
3. **Size calc:** `CalcTerrainSize()` aggregates brush bounds and entity key overrides → `mins`, `maxs`
4. **Vertex emit:** `EmitTerrainVerts()` loops over winding points, applies texture coordinates, deduplicates via `CompareVert()`
5. **Index generation:** Either `SideAsTristrip()` (preferred) or `SideAsTriFan()` (degenerate fallback) builds index buffers
6. **Normal computation:** `CreateTerrainSurface()` averages normals from all surfaces sharing each vertex → smooth lighting
7. **Output:** Populated `mapDrawSurface_t` added to global BSP surface list for later writing

## Learning Notes

**Offline-first design:** This module exemplifies why offline tools are separate from runtime—a map compiler can afford to:
- Allocate unbounded temporary structures (`realloc()` in growth chunks)
- Run expensive deduplication passes (`O(n²)` vert compare in `EmitTerrainVerts()`)
- Validate mesh topology exhaustively (degenerate triangle detection)

**Texture coordinate invariant:** The call to `QuakeTextureVecs()` with hardcoded `xynorm = {0, 0, 1}` assumes all terrain lies on the XY plane. This is a design constraint: terrain must be axis-aligned flats or height-mapped grids, not arbitrary 3D surfaces. Modern engines use more flexible patch-based or voxel terrain; Quake III's brush-based approach required this simplification.

**Color/alpha separation:** Vertex colors (`color[0/1/2]` = 255 for lighting; `color[3]` = alpha layer blend) uses the fourth byte for runtime blending. The rendering pipeline will interpret `color[3]` as a per-vertex blend factor between the two texture layers.

**No lighting computation here:** Unlike `cg_*/` or `server/` code, this file assembles geometry but leaves lighting to the main light-pass (`q3map/light.c`). The `// colors will be set by the lighting pass` comment signals this phase boundary.

## Potential Issues

- **Float-to-int tolerance:** The hardcoded `+ 0.1` offset in `floor()` (lines ~187, 272–273) is a hidden magic number that may cause precision issues if coordinates are intentionally fractional. No bounds checking on grid resolution.
- **Unbounded realloc in hot loop:** `EmitTerrainVerts()` reallocates indices in the main vert loop; could thrash memory on large terrains. Pre-allocation with a single size estimate would be more efficient.
- **No validation of alphamap dimensions:** `LoadAlphaMap()` only checks `>= 2x2`; no sanity check for grotesquely large images (e.g., 8K×8K) that could exhaust the hunk.
