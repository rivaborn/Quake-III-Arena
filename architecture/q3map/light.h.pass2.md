# q3map/light.h — Enhanced Analysis

## Architectural Role

This header defines the **lighting and tracing infrastructure for q3map**, the offline BSP compiler. It declares the central data structures and function signatures that drive the radiosity/lightmap generation pipeline during map preprocessing. Unlike runtime rendering (which consumes pre-computed lightmaps from BSP), this file orchestrates the **compile-time computation** of per-surface light samples and visibility through geometric ray tracing. The tracing subsystem (`light_trace.c`) is tightly coupled to this interface and provides the foundation for both direct lighting and radiosity iterations.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/light.c** — Main lighting engine; calls `TraceLine()`, reads/writes `surfaceTest[]`, manages `light_t` linked list
- **q3map/light_trace.c** — Implements `TraceLine()` and `PointInSolid()`; populates `trace_t` results; consumes `cFacet_t` and `traceWork_t`
- **q3map/lightv.c, q3map/lightmaps.c** — Consume the `lightmap_t` structure for per-surface light storage
- **q3map/mesh.c, q3map/misc_model.c** — Contribute facet geometry via `surfaceTest[]` array for tracing
- **q3map/map.c, q3map/bsp.c** — Build `light_t` instances from BSP entity definitions; define external lighting parameters (`lightscale`, `ambient`, etc.)

### Outgoing (what this file depends on)
- **common/mathlib.h, common/polylib.h** — Vector/plane math, winding polygon utilities for facet geometry
- **common/threads.h** — Distributed lighting computation across worker threads
- **q3map/shaders.h, q3map/mesh.h** — Shader properties (transparency for `cFacet_t.textureMatrix`) and mesh data structures
- **qcommon/bspfile.h** — BSP world geometry (via `light_trace.c` for collision queries)

## Design Patterns & Rationale

**Light-as-Linked-List**: Each `light_t` is a node in a `light_s *next` chain, rather than a flat array. This enables dynamic light addition during compile and efficient per-surface light queries without array bounds checks.

**Facet Subdivision**: The `cFacet_t` structure encodes both geometric (planes, points) and shading (texture matrix) data in a unified format. This allows the tracer to:
- Interpolate surface normals and texture coordinates at impact points (key for translucency and texture-space radiosity)
- Subdivide large patches/models into many small facets for sampling density

**Dual-Purpose Tracing**: `trace_t` holds both a **filter** (cumulative transparency from crossed surfaces) and a **hit point**. This supports:
- Soft radiosity through semi-transparent surfaces
- Early termination when filter becomes negligible

**WorkQueue Pattern**: `traceWork_t` is a temporary allocation passed to worker threads to avoid large per-frame stack allocations on older platforms (Win32 multi-threading, macOS). Each worker mutates its own `traceWork_t` without contention.

## Data Flow Through This File

1. **Compile-time initialization** (e.g., in `map.c`):
   - Parse entity definitions → create `light_t` instances (one per light entity or surface light)
   - Link all lights into the global `light_t *lights` chain
   - Set global scales (`lightscale`, `ambient`, `direct_scale`, etc.) from worldspawn

2. **Surface preparation** (in `mesh.c`, `misc_model.c`):
   - Tessellate patches and models into triangles
   - Wrap geometry in `cFacet_t` entries (bounding box, planes, points, texture matrix)
   - Store pointer in `surfaceTest[surfaceNum]`

3. **Radiosity iteration** (in `light.c`):
   - For each light and surface: allocate `traceWork_t` for this worker thread
   - Call `TraceLine()` from sample point to light
   - Accumulate `trace_t.filter` and `hitFraction` across bounces
   - Write result to `lightmap_t` for final image bake

4. **Lightmap finalization** (in `lightmaps.c`):
   - Flush accumulated `lightmap_t` samples to RAW texture format
   - Store as part of compiled BSP for runtime use

## Learning Notes

**Offline vs. Runtime Split**: Modern engines (Unreal, Unity) often blur compile-time and runtime lighting. Quake III keeps them strictly separate: **q3map computes** static lightmaps and radiosity once per map; the **renderer consumes** them each frame. This made sense in 2001 (pre-computed lightmaps were universal) but would be considered a limitation today.

**Geometric Ray Tracing for Radiosity**: The `cFacet_t` → `trace_t` → `filter` pipeline is a simplified radiosity solver. It's not full path-tracing (no importance sampling, limited bounces), but it was sufficient for Q3's visual style. Note the **texture-space radiosity** hint in `cFacet_t.textureMatrix` — this allowed per-pixel tracing rather than per-vertex, a detail-richness innovation at the time.

**Thread-Safe Facet Array**: The `surfaceTest[]` indexing is lock-free because it's populated once (during map init) before any worker threads read it. This is a good example of **compile-time parallelism patterns** where data is immutable after initialization.

**Shader Integration**: The `si` (`shaderInfo_s`) pointer in `light_t` suggests that shader properties (e.g., shader-emitted light models, translucency) feed directly into lighting calculations — not a separate rendering pass.

## Potential Issues

- **Fixed-Size Arrays**: `MAX_LIGHT_EDGES` (8) and hardcoded winding limits in `cFacet_t` could fail on complex geometry; no overflow checks visible here (likely in `light_trace.c`).
- **No LOD for Facets**: Large patches are subdivided into facets, but the subdivision density is static. Complex curved geometry could produce sparse facet coverage, leading to lighting artifacts.
- **Translucency Approximation**: `trace_t.filter` assumes linear opacity stacking, which breaks down for multiple overlapping semi-transparent surfaces or specular transmission.
