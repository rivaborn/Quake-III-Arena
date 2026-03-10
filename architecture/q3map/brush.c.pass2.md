# q3map/brush.c — Enhanced Analysis

## Architectural Role

This file is the **geometry manipulation engine** for the BSP compiler (`q3map`), an offline tool that converts `.map` files into compiled `.bsp` game worlds. It manages the complete lifecycle of brush geometry—from creation through spatial subdivision and final BSP tree integration. Since `q3map` is a build-time tool (not runtime), this code has no presence in the shipping engine; it exists only to prepare maps for consumption by the runtime renderer and collision systems.

## Key Cross-References

### Incoming (who calls this file)
- **q3map/map.c, qbsp.h**: Creates brushes from MAP entity/brush definitions; calls `AllocBrush`, `CreateBrushWindings`, `BrushFromBounds`
- **q3map/brush_primit.c**: Brush primitives operations; reuses brush allocation and manipulation
- **q3map/bsp.c** (BSP tree construction): Calls `FilterBrushIntoTree_r`, `BrushVolume` during tree recursion
- **q3map/tree.c** (spatial tree builder): Invokes brush filtering and splitting
- Brush windings feed into **q3map/writebsp.c** (final BSP serialization)

### Outgoing (what this file depends on)
- **qbsp.h** types: `bspbrush_t`, `side_t`, `winding_t`, `plane_t`
- **Global mapplanes array**: Plane database shared across q3map; indexed via `side->planenum`
- **Winding operations**: `BaseWindingForPlane`, `ChopWindingInPlace`, `FreeWinding` (from shared qbsp utilities)
- **Collision/geometric primitives**: Plane-side testing, point-in-bounds checks, area calculations
- **GLS_* graphics** (when `DrawBrushList` is active in debug builds): Link-time from editor/debug code

## Design Patterns & Rationale

**Memory Strategy**: `AllocBrush(numsides)` uses pointer-arithmetic sizing—the brush struct is variable-length with inline `sides[]` array. This avoids separate allocations per side but requires careful size calculation: `(int)&(((bspbrush_t *)0)->sides[numsides])` gives the full byte count. Paired with `FreeBrush` checks for winding pointers before freeing.

**Geometric Representation**: Brushes are defined by **half-space windings**. `CreateBrushWindings` generates a full winding for each side plane, then **clips** it against all other side planes via `ChopWindingInPlace`. This is idiomatic to Quake-era BSP tools—modern engines use explicit triangle meshes or convex hulls instead.

**Tree Integration**: `FilterBrushIntoTree_r` recursively descends the final BSP tree, **splitting** brushes at node planes and inserting copies into leaf nodes. Detail brushes (non-solid geometry for visual complexity) fragment across multiple clusters via `FilterDetailBrushesIntoTree`, while structural brushes remain whole per leaf.

**Epsilon Handling**: `PLANESIDE_EPSILON = 0.001` allows geometry to "slide by" minor floating-point discrepancies during brush-to-plane classification. This mirrors similar epsilon tolerances in runtime collision detection (e.g., `code/qcommon/cm_trace.c`).

## Data Flow Through This File

```
MAP file (entity/brush defs)
  ↓
map.c: parse → AllocBrush → CreateBrushWindings
  ↓
Brush geometry (sides + windings + bounds)
  ↓
bsp.c: recursively split & classify
  ↓
FilterBrushIntoTree_r (per leaf)
  ↓
Final BSP tree with brushes in leaves
  ↓
writebsp.c: serialize to .bsp file
```

**State tracked**: `c_active_brushes` counter (single-threaded mode) tracks allocations for debugging; `c_nodes` counts BSP tree nodes during compilation. Brush structs hold pointers to windings, which are mutable and freed when replaced or when brush is freed.

## Learning Notes

**Quake III geometry model**: Unlike modern engines using triangle meshes or signed distance fields, Q3 uses **convex polygon windings per plane**. This is computationally light for BSP splitting but geometrically inflexible and slow for runtime updates (hence brushes are baked offline).

**Offline vs. runtime divide**: Every function here serves the **compile-time** BSP pipeline. The runtime engine (`code/qcommon/cm_*.c`, `renderer/tr_bsp.c`) loads pre-built BSP files and never reconstructs brushes. This separation is stark: no brush code is linked into the shipped engine binary.

**Link-time integration**: `DrawBrushList` and debug output functions suggest this tool was debugged visually in an editor context (likely `q3radiant`), reusing OpenGL display calls. The `#include "qbsp.h"` pulls in shared tool infrastructure but **not** runtime headers like `qcommon.h`.

## Potential Issues

- **Memory fragmentation**: No memory pooling—each `AllocBrush` is a fresh `malloc`. On complex maps with thousands of brush fragments, this could fragment the heap.
- **Winding accumulation**: Temporary windings created by `ChopWindingInPlace` are freed, but if a plane chop fails mid-recursion, cleanup may be incomplete (no exception handling in C).
- **Epsilon accumulation**: Repeated plane-side checks with `PLANESIDE_EPSILON` may compound errors on deeply subdivided geometry; no guard against cascading misclassification.
- **Stack depth**: Recursive `FilterBrushIntoTree_r` mirrors BSP tree depth. Pathologically deep trees (e.g., linear chains of planes) could overflow the stack.
