# q3map/facebsp.c — Enhanced Analysis

## Architectural Role

This file implements the **face BSP tree builder**, a critical mid-stage component in the q3map offline BSP compilation pipeline. It converts a flat list of brush faces into a spatially-organized binary search tree, optimized for fast geometric queries during subsequent compilation phases. The output tree structure feeds into visibility (PVS) computation and geometry optimization stages downstream.

## Key Cross-References

### Incoming (dependencies on this file)
- **q3map tool driver** (`bspc.c` / other compilation stages) calls `FaceBSP()` to build the tree from structural/visible brush geometry
- **Visibility/PVS pipeline** likely consumes the resulting `tree_t` structure for cluster/portal computation
- **MakeStructuralBspFaceList()** and **MakeVisibleBspFaceList()** are entry points that pre-process brushes before tree construction

### Outgoing (what this file calls)
- **Geometric utilities** (`WindingOnPlaneSide`, `ClipWindingEpsilon`, `CopyWinding`, `FreeWinding`) — BSP/polygon operations from the shared q3map geometry library
- **Spatial indexing** (`FindFloatPlane`) — plane registry shared across compilation pipeline
- **Memory allocation** (`AllocNode`, `AllocBspFace`, `malloc/free`) — standard q3map allocators
- **Math utilities** (`VectorClear`, `VectorCopy`, `AddPointToBounds`) — from shared q_math

## Design Patterns & Rationale

1. **Recursive divide-and-conquer (BSP)**: `BuildFaceTree_r()` partitions the face list at each node using a heuristic split plane, mirroring the classic BSP tree algorithm. This enables logarithmic spatial queries in later stages.

2. **Multi-factor heuristic scoring** (`SelectSplitPlaneNum`):
   - Prefers planes that split the fewest faces (`splits` penalty)
   - Prefers planes aligned with more input faces (`facing` bonus)
   - Prioritizes axis-aligned planes (faster traces, smaller code)
   - Respects hint brushes (user-authored optimization hints get priority)
   - This greedy approach trades optimality for speed during compilation

3. **Block-boundary forcing**: Hard constraint at line 74–82 splits at 1024-unit block boundaries to maintain alignment with the engine's memory/spatial coherence model (`BLOCK_SIZE = 1024`).

4. **Lazy face list partitioning**: Faces are partitioned into `childLists[0/1]` inline; crossing faces are clipped on-demand during recursion, avoiding pre-splitting overhead.

## Data Flow Through This File

```
Input:
  bspbrush_t list (from BSP loader)
    ↓
  MakeStructuralBspFaceList / MakeVisibleBspFaceList
    ↓
  bspface_t linked-list + winding geometry

Processing:
  FaceBSP()
    → BuildFaceTree_r(node, face_list)
      → SelectSplitPlaneNum(node, list)  [greedy plane selection]
      → WindingOnPlaneSide() per face
      → ClipWindingEpsilon() for crossing faces
      → Recursively subdivide left/right children

Output:
  tree_t {
    headnode → node_t tree with:
      - node.planenum: split plane index (or PLANENUM_LEAF)
      - node.hint: whether split was hint-forced
      - node.children[0/1]: child nodes
      - node.mins/maxs: spatial bounds
  }
```

## Learning Notes

1. **Offline vs. online BSP**: This is an offline compiler tool, so it can afford expensive heuristics (greedy plane selection, full winding clips) that would be prohibitive in runtime geometry loading.

2. **Face BSP vs. World BSP**: The Quake III architecture maintains *two* distinct BSP trees:
   - **World BSP** (built by `bsp.c`): Solid geometry, used by collision system and renderer
   - **Face BSP** (this file): Face surface organization, used for visibility/PVS and geometry refinement

3. **Hint brush convention**: The `SURF_HINT` flag and `hint` field show a designer-friendly workflow where authors can mark strategic split planes to improve compilation quality—a common practice in early-2000s game engines before automatic optimization techniques matured.

4. **Axial plane preference** (line 124): Axis-aligned splits are preferred because they align with integer grid boundaries and allow faster AABB tests in downstream code. This is an era-appropriate optimization (SIMD wasn't standard for BSP in Q3A).

5. **Linked-list face representation**: Faces are stored in linked lists rather than arrays, reflecting C89 memory management constraints; modern engines would use growable arrays or scene graphs.

## Potential Issues

- **No balance heuristic**: The greedy heuristic does not attempt to balance tree depth; degenerate inputs (many coplanar faces) could produce deep linear trees. The `hintsplit` global variable and stateful `checked` field are also subtle state-management patterns.
- **Epsilon-based clipping** (`CLIP_EPSILON * 2`): The 2× multiplier suggests empirical tuning; face-clipping precision issues on thin geometry could cause cracks or t-junctions.
- **Memory leak risk**: Faces not freed during recursion (e.g., when `splitPlaneNum == -1`) are implicitly retained; the comment "List will be freed before returning" suggests external cleanup responsibility.
