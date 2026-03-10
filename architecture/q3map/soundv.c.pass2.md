# q3map/soundv.c — Enhanced Analysis

## Architectural Role

This file implements the **light volume propagation** subsystem for the offline BSP map compiler (`q3map`). It calculates how light travels through the compiled map geometry by tracing volumetric cones/pyramids from light sources, respecting portal/cluster boundaries and handling translucent surfaces. Unlike the runtime engine which uses precomputed lightmaps, `soundv.c` performs the offline radiosity computation that *generates* those lightmaps—bridging compile-time geometry (BSP faces, patches, shader properties) and runtime light delivery.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map main compilation pipeline** (`q3map.c`, `light.c`, `lightv.c`): calls volume computation routines to generate lightmap data for all drawable surfaces
- **Shader system** (`shaders.c`): provides `shaderInfo_t` at compile-time describing surface translucency and material properties; `soundv.c` reads these to determine light transmission
- **Mesh/patch detail layer**: accesses `detailMesh` from `lsurfaceTest_t` to compute per-lightmap-texel light contribution for curved surfaces
- **Thread pool** (`threads.h`, `mutex.h`): multithreaded light computation may lock surfaces via `lsurfaceTest_t.mutex` to avoid races

### Outgoing (what this file depends on)
- **Polylib** (`polylib.c`): winding allocation/clipping (`AllocWinding`, `ClipWindingEpsilon`, `WindingBounds`, `PointToPolygonFormFactor`)—foundational geometry ops
- **Light module** (`light.c`): exports calibration constants (`lightLinearScale`, `lightPointScale`, etc.) and form-factor calculation; `soundv.c` consumes these for attenuation models
- **BSP geometry** (via `bspfile.h`): reads cluster/portal topology, lightmap origins/vectors, and surface metadata from compiled BSP
- **Shader info** (`shaders.h`): translucency flags determine whether light penetrates or blocks in a volume
- **Qcommon mesh types** (`mesh.h`): `mesh_t` and `drawVert_t` for per-texel sampling on patches

## Design Patterns & Rationale

**Volume-based light tracing**: Rather than per-sample raycasting (expensive), the code traces planar volumes outward from point/area lights, clipping against BSP planes and portals. This mirrors the PVS-based visibility acceleration in the runtime renderer—clustering with portal gates avoids dense per-pixel computation.

**Translucent facet tracking**: The `transFacets[MAX_TRANSLUCENTFACETS]` array in `lightvolume_t` permits light to accumulate through semi-transparent geometry without terminating the volume, enabling realistic light bleeding through windows/gratings. This represents a deliberate tradeoff: some light can leak through materials while the volume remains active.

**Facet-level granularity**: Surfaces are decomposed into `lFacet_t` units with individual lightmap coordinates and texture matrices, allowing different areas of a curved patch to have distinct light values. This is more granular than the surface-level BSP design.

**Mutex-protected surfaces**: `lsurfaceTest_t.mutex` suggests parallel light computation across different light sources or surface subsets, critical for scaling to large maps without serialization bottlenecks.

## Data Flow Through This File

**Input**: BSP tree (clusters, portals, faces), shader definitions (translucency), light sources (`vsound_t` array), per-surface compile parameters (subdivision, face geometry).

**Transform**: 
1. Create light volumes as planar pyramids/cones emanating from light origins
2. Clip volumes against BSP portal/cluster boundaries using `VS_ChopWinding`
3. For each facet in a non-blocking surface, accumulate form-factor light using `PointToPolygonFormFactor`
4. Apply attenuation curves (quadratic/linear/constant; angle-based falloff)
5. Write final RGB triplet to lightmap array at correct texel offset

**Output**: Floating-point lightmap data (`lightFloats[]`) indexed by surface and texel, feeding the lightmap finalizer in `light.c`.

## Learning Notes

**Offline vs. online trade-off**: Q3A separates lightmap generation (expensive, done once offline) from runtime lighting (cheap lookup). `soundv.c` exemplifies this era's design—no dynamic lights in the BSP, only static precomputed radiosity. Modern engines compute GI at runtime via voxel cones or screen-space techniques.

**Portal/cluster reuse**: The runtime engine's PVS clusters and portals (used for visibility culling in `tr_world.c`) are repurposed here for light propagation—a space-efficient way to accelerate both systems without duplicating spatial structure.

**Form-factor geometry**: The call to `PointToPolygonFormFactor` ties into classical radiosity theory—the form-factor models how much of a light source's energy reaches a surface patch, accounting for distance and angle. This is foundational to offline GI computation predating prefiltered environment maps.

**Translucency as a first-class concern**: Unlike many early map compilers, Q3A's lightmapping pipeline explicitly models semi-transparent surfaces via `shader.translucent` flags in volumes, enabling creative lighting through glass and foliage.

## Potential Issues

- **Cluster bitmap overflow** (`clusterTested[MAX_CLUSTERS/8]`): If maps exceed 16384 clusters, this will silently corrupt memory; no bounds checking on cluster indices before bitset access.
- **Portal clipping precision**: Epsilon-based winding clipping may accumulate floating-point error in large maps with many portal clips; no error metrics or degenerate-face detection visible.
- **Translucent facet limit**: `MAX_TRANSLUCENTFACETS=32` is a hard cap; deeply nested glass/water may silently truncate light propagation if exceeded.
- **Single-pass volume**: Once a volume hits a non-translucent facet, it terminates; no refraction/reflection bounces, limiting realism for mirrors or curved glass.
