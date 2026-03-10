# q3map/lightv.c — Enhanced Analysis

## Architectural Role

This file implements **volumetric light propagation and visibility computation** for the offline BSP lighting compiler. It tracks how light spreads through the BSP world by computing light volumes—3D regions bounded by portal geometry—that represent directional or radial light rays as they traverse cluster/leaf boundaries. The module works in concert with `q3map/light.c` (surface lighting) to handle light blocking by translucent surfaces and complex geometric occlusion. As an offline tool component, it has no runtime presence.

## Key Cross-References

### Incoming (what calls this file)
- `q3map/light.c`: Main lighting driver calls volumetric functions to trace light propagation through portals; uses `lightvolume_t` structures and `VL_*` functions to determine light blocking across BSP boundaries
- Build system: Linked only into `q3map` tool binary (see `q3map/q3map.vcproj`, `Makefile`)

### Outgoing (what this file depends on)
- **BSP/Portal traversal**: Reads `lleaf_t` arrays (portal/cluster topology) populated by BSP loading in `q3map/light.c`
- **Shader system** (`q3map/shaders.h`): Accesses surface translucency/transparency flags to determine if light penetrates facets
- **Mesh structures** (`q3map/mesh.h`): Works with surface detail meshes to compute per-lightmap-pixel blocking
- **Threading** (`code/qcommon/{threads,mutex}.h`): Serializes multi-threaded access to `lsurfaceTest_t` via mutexes during parallel lighting passes
- **Winding/geometry** (`common/polylib.h`): Reuses winding clips and plane operations for light volume edge tests
- **Global state** (`extern` from `q3map/light.c`): `drawSurfaces[]`, `surfaceOrigin[]`, `samplesize`, shader info

## Design Patterns & Rationale

**Volumetric Light Tracing (vs. simple ray-casting)**
- Rather than cast rays individually, this module constructs 3D polyhedral volumes (`lightvolume_t`) that represent light "bundles" propagating through portal chains.
- Why: A single volume can block entire regions of a facet, avoiding per-pixel ray casts. Scales better when light must cross many portals.
- The volume is clipped against portals as it advances (`VL_ChopWinding`); translucent surfaces are accumulated rather than hard-blocked.

**Portal-driven visibility**
- Leverages the BSP portal graph (`lportal_t`, `lleaf_t`) already computed during BSP generation.
- Light volumes are initialized in a cluster and clip against portal planes as they traverse to neighboring leaves.
- Why: Reuses existing visibility data structure; avoids redundant world geometry queries.

**Facet-level detail**
- `lFacet_t` stores pre-computed lightmap coordinate bounds and texture matrices for each surface facet.
- Allows fast AABB intersection tests between light volumes and surface geometry.
- Why: Lightmaps are sparse; only facets "touched" by a light need checking.

**Deferred translucency handling**
- `lightvolume_t.transFacets[]` accumulates opaque surface indices encountered during traversal; blocking facet found via `facetNum`/`surfaceNum` fields.
- Contrast to immediate hard-blocking: allows light to weaken progressively through multiple translucent layers without splitting the volume each time.
- Why: Reduces volume fragmentation and trace overhead.

## Data Flow Through This File

1. **Initialization** (from `q3map/light.c`):
   - `lsurfaceTest[]` array populated with per-surface facet data; `vlights[]` array filled with light definitions
   - Portal/leaf topology already loaded from BSP

2. **Light Volume Creation**:
   - `VL_ConstructLight()` (stub in excerpt, likely in full file) creates a `lightvolume_t` with initial endpoint plane and bounding planes derived from the light's geometry (point or directed).

3. **Portal Traversal Loop**:
   - Volume clips against portal planes using `VL_ChopWinding()` and `ClipWindingEpsilon()`
   - Each portal transition updates `lightvolume_t.cluster` and records translucent surfaces crossed
   - Loop terminates when volume exits world or is fully consumed

4. **Facet Intersection**:
   - Volume's AABB checked against each `lFacet_t.x, .y, .width, .height` bounds
   - Intersection → light contribution computed on the lightmap using facet's texture/lightmap matrices

5. **Output**:
   - Accumulates RGB samples into `lightFloats[]` (the compiled lightmap data) at computed pixel coordinates

## Learning Notes

**Idiomatic to Q3A / Early 2000s Tools**
- **Explicit memory management**: Global arrays (`vlights[]`, `lsurfaceTest[]`) with fixed capacity (`#define MAX_LIGHTS 16384`); no dynamic allocation in hot paths.
- **Mutex-protected global state**: `lsurfaceTest_t.mutex` field is locked during parallel facet checks—no thread-local buffers; this reflects pre-TLS-era tool design.
- **Geometric precision**: Fixed-point epsilon tolerances (`ON_EPSILON 0.1`, `LIGHTMAP_PIXELSHIFT 0.5`) for plane clipping and lightmap-space snapping; contrast to modern normalized-texture-coordinate approaches.
- **Portal-graph-driven visibility**: Pre-computed portal/cluster hierarchy from BSP is the visibility oracle; no PVS-on-demand like runtime.

**Contrast to Modern Engines**
- Modern lightmap bakers (e.g., Enlighten, GPU-driven) directly rasterize world geometry into lightmap atlases; no "volume" abstraction.
- GPU compute shaders allow per-lightmap-pixel light tracing in parallel; CPU overhead is negligible.
- Bidirectional path tracing and photon mapping obsolete the "light volume" heuristic.

**Game Engine Concept Connections**
- **Shadow volumes** (Doom 3): Similar idea of using geometric volumes to represent light/shadow regions, though shadow volumes are rendered, not traced.
- **PVS portals**: Both shadow volumes and light volumes leverage explicit portal topology to partition space.
- **Lightmap baking**: This is the "offline" phase; at runtime, the prebaked lightmaps are applied via UV sampling in `tr_shade.c`.

## Potential Issues

- **Fixed buffer overflows**: `#define MAX_POINTS_ON_FIXED_WINDING 48` vs. `MAX_POINTS_ON_WINDING 64` — clipping operations may overflow if geometry is complex. Comment notes: "whenever this is overflowed parts of lightmaps might end up not being lit" — acknowledged but unresolved.
- **Thread safety**: `lsurfaceTest_t.mutex` is locked for facet tests, but no mutex protects writes to `lightFloats[]` itself; if two threads trace overlapping light volumes, results race. (Likely masked by task-level partitioning in parent `q3map/light.c`, but not enforced here.)
- **Memory footprint**: `lightFloats[MAX_MAP_LIGHTING]` can exceed 4 MB; combined with large surface/facet arrays, stack pressure is high. No mention of streaming or out-of-core handling.
