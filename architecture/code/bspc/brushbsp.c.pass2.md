# code/bspc/brushbsp.c — Enhanced Analysis

## Architectural Role
This file is the **core BSP tree builder for the BSPC offline compiler**, sitting at the crucial juncture between CSG (brush simplification) and AAS (bot navigation mesh). It is not part of the runtime engine; rather, it converts a flat list of raw brushes into a spatially coherent BSP tree that serves as the foundation for both **portal/visibility computation** and **AAS cluster and reachability analysis**. The generated `tree_t` flows downstream to the visibility pipeline (`code/bspc/portals.c`), which computes PVS, and to AAS creation (`code/bspc/be_aas_bspc.c`), which uses the BSP to subdivide space into walkable areas for bot pathfinding.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — Top-level main entry calls `BrushBSP` as the primary compilation stage after CSG
- **`code/bspc/csg.c`** — CSG output brush list is the input to `BrushBSP`
- **`code/bspc/portals.c`** — Consumes the resulting `tree_t` to compute portal connectivity and PVS
- **`code/bspc/be_aas_bspc.c`** — AAS compilation pipeline uses the BSP structure via `AAS_CalcReachAndClusters`

### Outgoing (what this file depends on)
- **`code/bspc/map.c`** — Reads global `mapplanes[]` array (all split plane definitions) and settings like `numthreads`, `create_aas`, `microvolume`
- **`code/bspc/aas_store.c` / `aas_create.c`** — Shared AAS memory and storage when `create_aas` flag is set (AAS-specific optimizations during tree building)
- **Winding utilities** — `BaseWindingForPlane`, `ChopWindingInPlace`, `ClipWindingEpsilon`, `WindingArea` — polygon clipping infrastructure
- **Thread primitives** — `ThreadLock`, `ThreadSemaphoreWait`, `AddThread`, `RemoveThread` from platform layer (`unix_shared.c` or `win32/win_shared.c`)
- **Memory allocator** — `GetMemory`, `FreeMemory`, `MemorySize` from `l_mem.h`

## Design Patterns & Rationale

### Dual-Path Execution (Recursive vs Iterative/Threaded)
`BuildTree_r` (recursive) and `BuildTree`/`BuildTreeThread` (iterative) implement two **BSP construction strategies**:
- **`BuildTree_r`**: Single-threaded, depth-first, stack-implicit. Simple and memory-efficient for small maps.
- **`BuildTree` + threads**: Iterative work-queue (stack or breadth-first queue selectable via `use_nodequeue`), enables **multi-core parallelism** by having multiple worker threads pull nodes from a global list. This is the **active path** (no production code calls `BuildTree_r` anymore).

**Rationale**: The iterative form decouples node processing from call-stack depth, allowing work stealing and load balancing across CPU cores — essential for large modern maps that would exhaust stack or be too slow on a single core.

### Greedy Plane Selection Heuristic
`SelectSplitSide` doesn't search globally for the optimal split; instead it **scores all candidate planes from the current brush set** using a cost function:
```
cost = 5*facing - 5*splits - abs(front-back) + [axial_bonus] - [epsilon_penalty]
```
**Rationale**: Globally optimal BSP construction is NP-hard; greedy per-level selection trades some tree quality for **O(n²)** compilation time. The scoring prioritizes:
- **Facing sides** (sides facing the camera in visible space, indicated by `SFL_VISIBLE`)
- **Minimizing splits** (fewer fragments = tighter clustering)
- **Balance** (splitting evenly reduces worst-case tree depth)
- **Axial planes** bonus (faster AABB tests at runtime)

### Epsilon Tolerance & Floating-Point Defense
`PLANESIDE_EPSILON = 0.001` allows brushes that barely touch a plane to "slide by" without splitting. This is a **numerical robustness pattern** — BSP construction is sensitive to FP precision; a vertex at `(100.0000, 200.0001, 300)` on a plane at `dist=200` should not cause a spurious split.

### Memory Tracking with Thread Guards
All memory stats (`c_brushmemory`, `c_active_brushes`, `c_nodememory`) are **only updated when `numthreads == 1`**. This avoids lock contention in multi-threaded builds; the stats become approximate but the code avoids serialization overhead.

## Data Flow Through This File

**Inputs** (from CSG / map.c):
- `bspbrush_t *brushlist` — One brush per CSG output; may overlap, contain complex geometry
- `vec3_t mins, maxs` — World bounding box (axis-aligned)
- `mapplanes[]` — Global pool of all split-plane definitions (normal + distance)

**Processing**:
1. Validate brushes, allocate root node, set volume brush (AABB)
2. Iterative/threaded loop: pick best split plane → partition brushes → create children
3. Base case: leaf node (no valid split) → classify content flags from brushes → optionally optimize for AAS

**Outputs** (to portals.c / AAS):
- `tree_t *tree` — Complete BSP tree rooted at `tree->headnode`
- Each node carries: plane ID, front/back children, brush list (internal nodes) or content flags (leaves)
- Optional: AAS area/cluster markers if `create_aas` is set

**State mutations**:
- **Free**: All input `brushes` are consumed/freed by `BuildTree` (no caller reuse)
- **Allocation spike**: Peak memory occurs when all brush fragments from all split operations are in-flight (tracked by `c_peak_brushmemory`)

## Learning Notes

### Idiomatic to Q3A BSP Compilation
- **Variably-sized struct trick** — `bspbrush_t` has a trailing `side[0]` array; `AllocBrush(numsides)` allocates `sizeof(bspbrush_t) + (numsides-1)*sizeof(side_t)` in a single block. This avoids pointer indirection and improves cache locality.
- **Side-as-split-candidate** — Each brush side is a candidate split plane. This ties the spatial structure to the input geometry; contrast modern engines that generate splits independently of brushes.
- **Content flags as leaf classification** — Leaves don't store geometry; only a bitmask of content flags (solid, water, lava, etc.). Collision detection queries these at runtime.
- **Visible vs non-visible distinction** — The `SFL_VISIBLE` flag marks sides that participate in final rendering; non-visible sides (detail brushes, internal geometry) are deprioritized in split selection.

### How This Differs from Modern Approaches
- **No voxel grid / octree** — Classic BSP is plane-aligned; modern engines often use axis-aligned grids or spatial hashing for faster queries.
- **Static, offline construction** — BSP is baked once at compile time; runtime dynamic objects don't affect it.
- **Greedy, not optimal** — No attempt to minimize tree depth or surface area in a global sense (would require expensive algorithms like SAH).
- **Geometry-driven splits** — The split planes come from brush surfaces, not from candidate generators like in modern BSP tools.

### Connections to Engine Concepts
- **Spatial indexing analogy to ECS**: If you view a BSP tree as a **spatial query accelerator**, it's conceptually similar to how a modern engine might maintain a spatial hash for broad-phase collision. The tree answers "what's in this region?" quickly via traversal.
- **Scene-graph parallel**: The BSP tree is Q3A's scene graph — it defines visibility and spatial coherence. Modern engines separate scene graphs (transform hierarchies) from spatial partitions; Q3 fuses them in the BSP.
- **Leaf content as ECS tags**: The content flags on a leaf are like ECS tags (solid, water, etc.), determining which systems process that region.

## Potential Issues

1. **Memory scaling with thread count**: `c_brushmemory` tracking is disabled in multi-threaded builds (`numthreads > 1`), so peak memory statistics are unreliable for parallel compiles. Not a correctness bug, but limits observability.

2. **Split plane quality under parallelism**: When multiple threads create children concurrently, they independently call `SelectSplitSide` on their brush subsets. A side that's optimal at the local level may not be optimal for the global tree. Single-threaded `BuildTree_r` doesn't have this issue (deterministic greedy selection).

3. **Floating-point sensitivity in `BoxOnPlaneSide` fast path**: The `signbits` lookup assumes `plane->signbits` is correctly precomputed. If a plane's normal is not unit-length or contains NaNs, the 8-case switch can misbehave. The code assumes `mapplanes[]` is well-formed; no validation is performed.

4. **AAS mode partial cleanup**: When `create_aas` is true, `FreeBrushList` is called early during `BuildTree`, but some brush data may still be referenced by AAS structures (`node->volume` brush). The code appears safe (AAS snapshot is taken before free), but tight coupling here.
