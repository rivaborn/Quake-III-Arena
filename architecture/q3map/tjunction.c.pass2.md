# q3map/tjunction.c — Enhanced Analysis

## Architectural Role

This file implements **T-junction resolution** for the offline BSP map compiler (`q3map/`). T-junctions occur when a patch or surface edge intersects a brush edge at a non-vertex point, causing lighting discontinuities and rendering artifacts. The module preprocesses all map surfaces to insert vertices at these intersection points, subdividing surfaces into valid triangle strips before lightmap allocation and final BSP construction.

## Key Cross-References

### Incoming (who depends on this file)
- `FixTJunctions()` is the sole public entry point, called from the main BSP compilation pipeline (likely `q3map/bsp.c` or similar) after surface pruning but **before lightmap allocation**
- Processes all `mapDrawSurface_t` entries in a global surface list (inferred from `FixSurfaceJunctions()` iteration pattern)

### Outgoing (what this file depends on)
- **Math utilities** (`q_math.c`): `VectorSubtract`, `DotProduct`, `VectorNormalize`, `VectorCopy`, `VectorScale`, `VectorLength`, `MakeNormalVectors`
- **Error handling** (`qcommon/common.c`): `Error()` for fatal conditions (overflow of global arrays)
- **Memory**: Standard `malloc()`/`free()` for linked-list node allocation
- **I/O**: `qprintf()` for diagnostic output (line count, rotation statistics)

## Design Patterns & Rationale

**Circular Doubly-Linked List** (`edgePoint_s`): Points on each edge are maintained in sorted order (by parametric intercept). The sentinel node (`e->chain`) simplifies insertion logic and boundary checks.

**Plane-Based Edge Representation** (`edgeLine_t`): Each edge is defined by:
- Two perpendicular plane equations (normal + distance) to rapidly classify if a vertex lies on the edge
- Origin + direction vector for computing intercept values
This avoids expensive 3D distance calculations during insertion.

**Epsilon Tolerance**: `LINE_POSITION_EPSILON` (0.25) and `POINT_ON_LINE_EPSILON` (0.25) account for floating-point snap rounding in the geometry (comment: "plus SNAP_INT_TO_FLOAT"). This is a typical pattern in offline geometry tools.

**Surface Rotation Heuristic** (lines ~420–460): After subdividing a surface, the code rotates vertices to ensure the first vertex is surrounded by an unsplit original edge. This enforces a valid triangle-fan/strip topology (note: commented-out code for centroid-fan generation suggests an alternate strategy was considered).

## Data Flow Through This File

1. **Initialization**: Global `edgeLines[]` and `originalEdges[]` arrays populated
2. **Edge Tracking**:
   - `AddSurfaceEdges()` → for each surface edge, call `AddEdge()` 
   - `AddEdge()` tests if edge collinear with existing line or creates new line; stores reference in surface's lightmap[0] field
   - `AddPatchEdges()` adds colinear-border edges from patches (bridges some patch↔brush T-junctions)
3. **Surface Subdivision**:
   - `FixSurfaceJunctions()` iterates original surface vertices
   - For each edge, walks the sorted point chain and inserts new vertices where t-junction points fall
   - Reconstructs surface with interpolated normals and texture coordinates
4. **Output**: Modified surface with `numVerts` increased; original vertices may be rotated

## Learning Notes

**Offline vs. Runtime**: This is **not** executed at game runtime. The subdivided surfaces are baked into the final BSP and PVK3. This reflects Quake 3's era—modern engines use runtime tessellation or LOD pipelines.

**Floating-Point Robustness**: The repeated epsilon-based comparisons and sorting by intercept illustrate careful handling of geometric numerics. Lines ~89–105 show the standard pattern: compute parameter, test tolerance, insert if unique.

**Mesh Topology Constraints**: The surface rotation logic reveals that Q3 surfaces must satisfy specific winding/fan constraints. This is a vestige of fixed-function OpenGL strip/fan primitives (see `renderer/tr_surface.c` for how these surfaces are fed to the GPU).

**Hybrid Fixed/Adaptive Approach**: `AddPatchEdges()` and `FixSurfaceJunctions()` together form a multi-stage fix: colinear border edges handle simple cases, then full edge-tracking handles complex patch↔brush intersections. This is more surgical than a global retriangulation.

## Potential Issues

- **Global Array Overflow**: `MAX_EDGE_LINES` (0x10000) and `MAX_ORIGINAL_EDGES` (0x10000) are hard limits; very complex maps could hit these (`Error()` is fatal)
- **Numerical Stability**: Heavy reliance on floating-point intercept computation; accumulation errors could cause points to diverge from colinearity in large maps
- **Incomplete Fix**: Comment at line ~249 flags that some patch↔brush T-junction classes may still escape detection (rely on `AddPatchEdges()` workaround instead)
