# q3map/portals.c — Enhanced Analysis

## Architectural Role

This file implements the **portal generation stage** of the Quake III BSP compiler (`q3map`), a completely offline tool. Portals are the boundaries between BSP leaf nodes—convex polygons that partition space and define node adjacency. After the BSP tree is built via plane splitting, portals are created and clipped to encode which leaves are connected, forming the input for later PVS (potentially visible set) and VIS processing. The generated portals are written into the compiled `.bsp` file and used at runtime by `qcommon/cm_*.c` (collision world) and `code/botlib/be_aas_*.c` (navigation) to answer spatial queries.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map/qbsp.h** defines `portal_t`, `node_t` structures and function signatures
- Called directly by **q3map/tree.c** (BSP tree recursive subdivision) which invokes `MakeTreePortals_r()` after all splitting is complete
- Portal data is ultimately serialized into `.bsp` lump data by **q3map/writebsp.c**

### Outgoing (what this file depends on)
- **common/mathlib.h** / **code/qcommon/qcommon.h** for `VectorCopy`, `VectorSubtract`
- **q3map/poly.h** (via qbsp.h): `BaseWindingForPlane()`, `ChopWindingInPlace()`, `ClipWindingEpsilon()`, `FreeWinding()`, `WindingIsTiny()`—core winding (convex polygon) clipping primitives
- **code/qcommon/cm_polylib.h** for polygon math (boundary winding construction)
- Global `mapplanes[]` array (from **q3map/map.c**): all split planes in the BSP tree

## Design Patterns & Rationale

### 1. **Plane-Clipping Pipeline**
`BaseWindingForNode()` and `SplitNodePortals()` use **iterative plane clipping**. Each portal winding starts as a full-plane rectangle, then is clipped by all ancestor split planes to restrict it to the parent's volume. This is mathematically correct for BSP trees: a node's portal boundaries are the intersection of all ancestor half-spaces. The epsilon values (`BASE_WINDING_EPSILON`, `SPLIT_WINDING_EPSILON`) prevent numerical precision errors from creating degenerate geometry.

### 2. **Doubly-Linked Node Lists**
Portals maintain two linked lists—one for each adjacent leaf (`portal->next[0]`, `portal->next[1]`). This enables fast iteration during `SplitNodePortals()` without building separate data structures. The cost is that `RemovePortalFromNode()` must linear-search the list to unlink, but portal count per leaf is typically small.

### 3. **Lazy Allocation & Peak Tracking**
The global counters (`c_active_portals`, `c_peak_portals`, `c_tinyportals`) are updated only when `numthreads == 1`, avoiding contention in multithreaded builds. This is a pragmatic tradeoff—accurate statistics are sacrificed for thread safety.

### 4. **Tiny Portal Elimination**
Portals too small to be meaningful (checked by `WindingIsTiny()`) are discarded early. However, their reference points are stored in nodes to prevent completely unreachable volumes. This allows AAS generation (pathfinding) to gracefully degrade rather than crash.

## Data Flow Through This File

```
Input:  BSP tree with split planes (mapplanes[])
        Root node points to outside_node
        
Step 1: MakeHeadnodePortals() — Create 6 axis-aligned bounding portals
        around the entire map volume; clip each by the other 5

Step 2: MakeNodePortal() — For each internal node:
        - Get base winding from plane
        - Clip by all parent planes (BaseWindingForNode)
        - Clip by all sibling portals at this node
        - Allocate if non-tiny; store portal.hint flag

Step 3: SplitNodePortals() — When node splits into children:
        - Iterate portals attached to the parent node
        - Clip each by the split plane (ClipWindingEpsilon)
        - Reattach halves to children
        - Discard tiny fragments; track reference points

Step 4: CalcNodeBounds() — Compute node AABB from portal windings
        (used for frustum culling in vis phase)
        
Output: BSP tree with portal chains at each node
        Written to .bsp file lump
        Later consumed by q3map/vis.c (PVS calculation)
        Runtime used by CM and AAS systems
```

## Learning Notes

### Portal Semantics in Quake III
- **Unlike modern engines**, Quake III portals are **implicit edges** of the BSP partition. They define adjacency but don't store explicit PVS information themselves—that's computed later by the **VIS** phase (`q3map/vis.c`), which flood-fills connected portals to compute cluster visibility.
- **Portal windings** are stored only during compilation. The runtime BSP (`code/qcommon/cm_*.c`) stores only plane indices and leaf flags; portals are reconstructed on-the-fly during sweep traces.
- **The "outside_node"** is a special infinite leaf outside the map. All boundary portals connect to it. This is crucial for flood-fill VIS: the outside is always visible, so any leaf connected to outside gets a direct PVS line.

### Era-Specific Idiomatic Patterns
1. **No dynamic structures**: Portals are allocated with `malloc()` and manually freed. No modern pool allocators, no memory arenas beyond the hunk buffer.
2. **Stateless functional design**: Portal operations are nearly pure (apart from the global counters). This makes the BSP generation deterministic and easy to debug.
3. **Epsilon tolerances**: Numerical robustness is achieved via explicit epsilon constants, not automatic precision handling. Windings that degenerate below epsilon are discarded entirely.
4. **Explicit linked lists**: No intrusive containers or helper macros—just raw `next` pointers and manual list manipulation.

### Modern Engines Contrast
Modern engines (Unreal, Godot) either:
- Use spatial hashing or BVH trees instead of explicit portals for visibility
- Compute PVS offline as a dense bit matrix in a preprocessing pass
- Or skip PVS entirely and rely on occlusion culling at runtime

Quake III's approach is elegant for its era but trades memory (storing all portal windings) for simplicity and predictability.

## Potential Issues

### 1. **Unbounded Portal Warnings** (lines ~450–460)
The code warns if nodes have coordinates outside `MIN_WORLD_COORD` / `MAX_WORLD_COORD` (typically ±131072). This can happen if brushes exceed map bounds. The engine will clamp traces, but unbounded geometry may produce unexpected visibility. **Risk:** Authored maps with extreme geometry might exhibit PVS holes or visible seams.

### 2. **Recursive Depth Limit**
`MakeTreePortals_r()` is recursive with no depth guard. A degenerate BSP tree with all splits along one axis could recurse to depth ≥ 65536, exhausting stack. **Risk:** Malformed maps from buggy compilers or hand-editing could crash.

### 3. **Numeric Precision**
`ChopWindingInPlace()` clips by planes with floating-point comparisons. If many planes are nearly coplanar (e.g., thin corridor with many faces), epsilon accumulation could cause windings to vanish prematurely. **Mitigation:** The `SPLIT_WINDING_EPSILON` is tuned empirically; re-tuning it could break existing maps.

### 4. **Memory Leak on Unreachable Code Paths**
If `WindingIsTiny()` returns true but memory cleanup is skipped, or if `ChopWindingEpsilon()` returns inconsistent results (both `NULL`), portals and windings might leak. The code appears sound, but the lack of a resource-guard pattern (RAII) makes manual audits necessary.
