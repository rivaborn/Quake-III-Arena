# q3map/mesh.c — Enhanced Analysis

## Architectural Role

This file is a critical mesh processing utility for the **q3map BSP compiler**. It handles subdivision and optimization of curved surfaces (Bezier patches) that mappers place in Q3 levels. The adaptive subdivision converts mathematically-defined smooth curves into tessellated triangle meshes that can be incorporated into the BSP tree, enabling curved surfaces without the memory cost of high-resolution static geometry.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/patch.c** — Likely the primary consumer; calls `SubdivideMesh` to tessellate Bezier patches during BSP surface collection phase
- **q3map/mesh.h** — Public header exposing the mesh API
- Other q3map surface processors (brush.c, surface.c) may use these utilities for mesh manipulation during compile phases

### Outgoing (what this file depends on)
- **qcommon/q_math.c** — `VectorLength`, `VectorNormalize`, `CrossProduct`, `DotProduct`, `VectorMA`, `VectorSubtract`, etc.
- **libc** — `malloc`, `free`, `memcpy`, `memmove` for memory and data movement
- **stdio** — `_printf` for debug output (only in `PrintMesh`)
- No engine subsystem dependencies; purely offline computational tool

## Design Patterns & Rationale

**Adaptive Error-Based Subdivision**: `SubdivideMesh` uses a classic technique: repeatedly split edges if the midpoint diverges by more than `maxError` from the quadratic interpolated position, or if segment length exceeds `minLength`. This balances visual fidelity against BSP complexity — high-curvature patches get finely tessellated; flat regions remain coarse.

**Wrapping Mesh Support**: `MakeMeshNormals` detects and handles wrap-around edges (closed U/V axes), allowing seamless tiling patches. This enables mappers to create smooth cylindrical and toroidal surfaces.

**In-Place Array Expansion**: `SubdivideMesh` uses a fixed-size 2D array (`expand[MAX_EXPANDED_AXIS][MAX_EXPANDED_AXIS]`) as a workspace, dynamically growing the mesh width and height while preserving the array structure. Offsets into `originalWidths`/`originalHeights` track which control points correspond to subdivided midpoints vs. original vertices.

**Per-Vertex Normal Computation**: `MakeMeshNormals` samples 8 neighboring vertices around each point to compute an averaged normal, handling edge cases by wrapping or early-termination at mesh boundaries.

## Data Flow Through This File

1. **Input**: A `mesh_t` with `width×height` control point grid (from Bezier patch definition in BSP entity data)
2. **Subdivision Phase** (`SubdivideMesh`):
   - Iteratively insert new columns/rows where curvature exceeds error tolerance
   - Linearly interpolate new vertices between adjacent control points
   - Produces a finer mesh (possibly 2–4× larger if highly curved)
3. **Optimization Phase** (`RemoveLinearMeshColumnsRows`):
   - Optionally removes redundant columns/rows where all vertices lie on a straight line
   - Reduces final BSP surface count
4. **Normal Generation** (`MakeMeshNormals`):
   - Computes per-vertex surface normals via cross-product of neighboring edge vectors
   - Used during lighting calculation and back-face culling in BSP
5. **Output**: A tessellated mesh ready for BSP face insertion and lighting

## Learning Notes

- **Bezier Patch Handling**: Quake III uses quadratic Bezier surfaces. The midpoint error formula `(p0 + 2*p1 + p2) * 0.25 - midpoint` catches deviation from the true curve.
- **Constraint Stack Allocation**: The `expand[MAX_EXPANDED_AXIS][MAX_EXPANDED_AXIS]` array is stack-allocated; very large or complex patches could overflow or hit MAX_EXPANDED_AXIS limits (typically ~32 or 64 vertices per dimension).
- **Deterministic Mesh Generation**: Wrapping logic and sequential subdivision ensure reproducible results across different systems — critical for BSP reproducibility.
- **Modern Engines**: Most current engines would either (a) subdivide at runtime in shaders, or (b) bake high-res static meshes offline. Quake III's compile-time approach balances static memory usage against BSP complexity.

## Potential Issues

- **Stack Overflow Risk**: If a map contains a Bezier patch with extremely high curvature, iterative subdivision could exceed `MAX_EXPANDED_AXIS`. Early termination prevents crashes but may produce visually incorrect geometry.
- **Wrapping Edge Case**: `MakeMeshNormals` wrapping logic can be brittle for edge-adjacent vertices; degenerate normals are silently clamped to `[0, 0, 0]` rather than reported.
- **No Input Validation**: Functions assume `mesh_t` validity; malformed control point data (NaN, Inf) will propagate through computations.
