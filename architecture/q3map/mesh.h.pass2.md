# q3map/mesh.h — Enhanced Analysis

## Architectural Role

This file is a **compile-time mesh utility layer** for the BSP map compiler tool (`q3map/`), not part of the runtime engine. It provides data structures and algorithms for manipulating geometric mesh data during map preprocessing—specifically for subdividing Bézier patches, transforming vertices, and computing surface normals that eventually get baked into the compiled BSP format. Unlike runtime geometry (which exists in the renderer's vertex buffers and shader systems), `mesh.h` operates on intermediate mesh representations during the offline compilation pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/*.c** — Other offline compiler modules (mesh.c, patch.c, surface.c, etc.) import these mesh manipulation functions and the `mesh_t` type
- **Tool-only usage** — No runtime engine subsystems (renderer, qcommon, cgame, game VM) reference these functions; the mesh data is consumed and converted into final BSP lump data during compilation

### Outgoing (what this file depends on)
- **q_shared.h / q_math.h** — Basic math utilities (`vec3_t`, vector operations) and foundational types
- **tr_types.h** — References `drawVert_t` (draw vertex structure), which is defined in the renderer but reused as the vertex type during mesh compilation
- **No runtime engine dependencies** — Does not import qcommon, renderer, or server APIs; remains isolated to the offline toolchain

## Design Patterns & Rationale

- **Simple functional C API**: No object lifecycle management; utility functions operate on stack-allocated `mesh_t` + heap-allocated vertex arrays
- **Copy-on-modify pattern**: `CopyMesh`, `TransposeMesh`, `InvertMesh` create new mesh instances rather than modifying in-place — reflects the immutable-intermediate-data ethos of offline compilation
- **Subdivision on demand**: `SubdivideMesh` and `SubdivideMeshQuads` dynamically tessellate high-curvature Bézier patches to reduce geometric errors in the final BSP
- **Pre-computed axis tracking**: `originalWidths`/`originalHeights` arrays track source mesh dimensions across the compilation pipeline, likely for memory pooling or validation
- **No streaming/cleanup overhead**: Tool runs once; no need for incremental deallocation or memory defragmentation as seen in the runtime engine

## Data Flow Through This File

1. **Input**: Raw mesh geometry from `.ase` (ASCII Scene Export) model loaders or Bézier patch definitions extracted from the BSP entity strings during map parsing
2. **Transformation**: 
   - `SubdivideMesh` tessellates curved surfaces based on curvature/edge-length thresholds
   - `TransposeMesh`/`InvertMesh` reorient geometry for collision/lighting calculations
   - `MakeMeshNormals` computes per-vertex normals for shading
   - `PutMeshOnCurve` snaps vertices to Bézier surface boundaries
3. **Output**: Flattened vertex/index data written into BSP lump structures (surface definitions, lightmaps, model geometry)

## Learning Notes

- **Offline-first mentality**: Contrasts sharply with the runtime renderer's constraint of real-time vertex processing. The compiler can afford O(n²) subdivision and expensive normal recomputation.
- **Bézier patch handling**: The `Max*` #defines and subdivision functions reflect Q3A's use of curved surfaces for both geometry and terrain—a high-end feature for 2001 hardware that required aggressive compile-time preprocessing.
- **Shared vertex type across pipeline**: Using `drawVert_t` (a runtime renderer struct) in an offline tool shows tight coupling—the compiler must know the exact vertex layout the renderer expects.
- **Not idiomatic to modern engines**: Modern engines typically decouple offline mesh processing (using Assimp, Mikktspace, etc.) from the runtime vertex format; Q3A's direct use of `drawVert_t` in q3map reflects its monolithic, single-tool design.

## Potential Issues

- **No input validation**: The functions assume valid `mesh_t` and non-NULL vertex pointers; malformed input (negative dimensions, NULL verts) could cause buffer overruns or segfaults during subdivision
- **Fixed-size global arrays**: `originalWidths`/`originalHeights` capped at `MAX_EXPANDED_AXIS` (128) with no bounds checking; overly complex maps could silently overflow
- **Floating-point precision**: `SubdivideMesh` compares distances to `maxError` threshold; accumulated floating-point error in recursive subdivision could produce unexpected results on edge cases
