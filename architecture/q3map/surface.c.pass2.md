# q3map/surface.c — Enhanced Analysis

## Architectural Role

This file is the **draw surface construction and BSP tree integration module** of the offline q3map BSP compiler. It converts brush-side geometry into `mapDrawSurface_t` render primitives, applies texture transformations, subdivides surfaces based on shader properties, and spatially links them into the compiled BSP tree so the runtime renderer can efficiently retrieve visible surfaces during frame rendering. It bridges the gap between geometric brush compilation and the runtime rendering system.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map main pipeline** (`q3map.vcproj`, `bsp.c`) — calls `ClipSidesIntoTree` and `SubdivideDrawSurfs` as part of the main BSP compilation sequence (after BSP tree construction, before finalization)
- **Brush processing** (`brush.c`, `map.c`) — creates brush sides that feed into `DrawSurfaceForSide`
- **Shader system** (`shaders.c`) — provides `shaderInfo_t` with subdivision and surface flags consumed here

### Outgoing (what this file depends on)
- **qbsp.h / qbsp structures** — `mapDrawSurface_t`, `bspbrush_t`, `side_t`, `winding_t`, `shaderInfo_t`
- **BSP tree** (`tree.c`) — reads node/leaf structure; writes `drawSurfRef_t` references into leaf nodes
- **Winding operations** (`q3map/`-local utilities) — `ClipWindingEpsilon`, `ReverseWinding`, `CopyWinding`, `FreeWinding`, `WindingFromDrawSurf`
- **Math utilities** — `DotProduct`, `VectorCopy`, bounds operations, `ComputeAxisBase` for texture matrix
- **Collision/plane utilities** — `mapplanes` global array for plane normals/distances
- **Memory/output** — `malloc`, `qprintf` for logging

## Design Patterns & Rationale

**Pool Allocation:** Global `mapDrawSurfs[MAX_MAP_DRAW_DRAW_SURFS]` + counter avoids fragmentation during compilation and makes final export trivial (contiguous array write).

**Recursive Tree Traversal:** `ClipSideIntoTree_r` and `FilterMapDrawSurfIntoTree` recursively descend the BSP tree. This follows the pattern of all BSP operations in this era: spatial recursion is simpler and more cache-friendly than top-down iteration.

**Dual Texture Coordinate Modes:** The brush primitive vs. old-style `vecs` branching (`if (g_bBrushPrimit==BPRIMIT_OLDBRUSHES)`) reflects the evolution of the Quake III editor. Brush primitives introduced a matrix-based texture space; legacy maps used per-plane texture vectors. Both must be supported during compilation.

**Polygon Clipping & Visibility:** `ClipSideIntoTree_r` computes the visible hull of each side by recursively clipping against all non-opaque clusters. This is classic BSP visibility: only the parts touching non-opaque leaves matter for rendering.

**Late Subdivision:** `SubdivideDrawSurfs` runs *after* tree construction and side visibility computation. This avoids splitting sides needlessly and ensures only visible fragments are subdivided.

## Data Flow Through This File

1. **Input:** Brush sides with windings, shader info, compiled BSP tree (nodes/leaves with opaque flags)
2. **DrawSurfaceForSide:**
   - Allocates a `mapDrawSurface_t` in the global pool
   - Snaps vertex positions to a grid (precision control)
   - Computes per-vertex s/t texture coordinates (either brush-primitive or legacy mode)
   - Tracks min/max texture bounds
   - Normalizes texture coordinates relative to shader dimensions
   - Stores references to brush, side, and shader
3. **ClipSidesIntoTree:**
   - For each brush side, copies its winding
   - Recursively clips against BSP planes via `ClipSideIntoTree_r`
   - Keeps only fragments in non-opaque leaves (the "visible hull")
   - Creates final draw surface from visible hull
   - Optionally creates a back-side for fog volumes
4. **SubdivideDrawSurfs:**
   - Scans all draw surfaces created so far
   - If shader has subdivision > 0, recursively clips surface into a grid
   - Replaces oversized polygons with smaller tiles (reduces runtime GPU batching cost)
5. **FilterMapDrawSurfIntoTree:**
   - Scans mesh vertices or sample points of each surface
   - For each point, descends the BSP tree and marks all non-opaque leaves it touches
   - Stores `drawSurfRef_t` in leaf nodes so renderer knows which surfaces to draw

## Learning Notes

**BSP-Era Rendering Pipeline:** This code reflects the pre-modern engine design where offline compilation and runtime visibility are tightly coupled. The BSP tree *itself* is the spatial index; there's no separate acceleration structure. Every leaf must maintain a list of potentially-visible surfaces.

**Texture Space Dual-Mode:** The `g_bBrushPrimit` branch shows how a shipping engine must often support multiple content pipelines. Newer maps use brush primitives (more intuitive in editors); legacy Q3 maps use planar texture vecs. Both coexist in the same compiler.

**Polygon-Centric Geometry:** Unlike modern engines (which use meshes and materials), Q3 surfaces are individual polygons with per-vertex data (`drawVert_t` with xyz, st, normal). This reflects the era's shader-per-surface granularity.

**Winding as Primary Representation:** Windings (ordered vertex lists) are the canonical polygon form in the offline tool. They're clipped, reversed, and traversed frequently. Modern tools prefer triangle/index buffers from the start.

**Grid Subdivision:** The `SubdivideDrawSurfs` technique (splitting large surfaces into a grid) is a pre-GPU optimization. GPUs later removed this constraint, but Q3 maps still contain pre-subdivided surfaces from compilation.

## Potential Issues

**Incomplete Merge Implementation:** `MergeSides()` is declared but contains only logging—no actual merging occurs. The loop body is commented out. This was likely planned for batching sides but left unfinished.

**No Subdivision Caching:** If a shader has very fine subdivision (e.g., `subdivisions = 4`), recursive `SubdivideDrawSurf` calls could explode combinatorially. No memoization guards against this; deeply nested recursion could overflow the stack on pathological maps.

**Texture Coordinate Normalization:** The adjustment `dv->st[i] -= mins[i]` assumes `globalTexture` is false. If true, bounds are computed but discarded. If a very large surface spans far-out texture space, this could lose precision (no re-normalization for `globalTexture`).

**No Bounds Check on Winding Points:** `w->numpoints > 64` is checked in `DrawSurfaceForSide`, but other functions receive windings from clipping operations which could theoretically exceed 64 points if many planes clip them.
