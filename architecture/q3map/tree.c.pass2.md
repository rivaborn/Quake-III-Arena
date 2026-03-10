# q3map/tree.c — Enhanced Analysis

## Architectural Role

This file manages the lifecycle of the **BSP tree data structure** during offline map compilation in `q3map/`, the command-line BSP compiler tool. It provides spatial partitioning tree operations—tree node location, recursive teardown, and portal/connectivity cleanup—that sit at the junction between the geometric construction pipeline (which builds the tree) and memory management. The tree is the primary intermediate representation in BSP compilation: it organizes map geometry into a recursive spatial hierarchy before being serialized into the final `.bsp` file format.

## Key Cross-References

### Incoming (who depends on this file)

- **`q3map/` compilation pipeline** calls `FreeTree()` and `NodeForPoint()` as part of phase transitions:
  - Likely called from `q3map/bsp.c` or `q3map/tree.c`'s callers during BSP construction teardown
  - `NodeForPoint()` used for geometric queries during tree construction
- **`q3map/` portal/visibility system** calls `RemovePortalFromNode()` indirectly through `FreeTreePortals_r()` to unlink portal doubly-linked lists
- **Debug/logging subsystem** may call `PrintTree_r()` for visualization of tree structure during development

### Outgoing (what this file depends on)

- **`qcommon/q_math.c`**: `DotProduct()` for point-to-plane classification in `NodeForPoint()`
- **Memory allocators** (defined elsewhere in q3map): `FreePortal()`, `FreeBrushList()`, `FreeBrush()` for releasing nested structures
- **Global symbols** (q3map scope):
  - `extern c_nodes`: statistics counter tracking total node count (read/written)
  - `extern mapplanes[]`: plane data indexed by `node->planenum`
  - `numthreads`: threading configuration affecting cleanup instrumentation
- **`qbsp.h`**: Common BSP compiler definitions including `node_t`, `tree_t`, `portal_t`, `plane_t` structures and `PLANENUM_LEAF` sentinel

## Design Patterns & Rationale

### Recursive Tree Traversal with Post-Order Cleanup
All three recursive functions (`FreeTreePortals_r`, `FreeTree_r`, `PrintTree_r`) follow post-order traversal: **children processed before parent**. This ensures:
- Portals are unlinked from child nodes before the child is freed (`FreeTreePortals_r` runs first)
- Child nodes are freed before parent nodes (no dangling parent pointers)
- Print output flows from leaves to root (readable indentation)

This is the idiomatic pattern for BSP tree structures, which are fundamentally binary (two children per non-leaf node).

### Forward Declaration of Portal Removal
`RemovePortalFromNode()` is declared but not defined here—it lives elsewhere (likely `q3map/portals.c` or similar). This reflects **circular dependencies in the q3map subsystem**: portals reference nodes, nodes reference portals, so the cleanup sequence is split across files:
- This file handles **node-side** cleanup (recursion, bookkeeping)
- Portal file handles **portal-side** cleanup (bidirectional unlink)

The pattern mirrors real-world game engines: scene graph teardown (`FreeTree_r`) separate from visibility/portal teardown.

### Conditional Thread Accounting
```c
if (numthreads == 1)
    c_nodes--;
```
Only decrements `c_nodes` in single-threaded mode. This suggests **multi-threaded q3map exists** (parallel BSP construction), where thread-local accounting differs from global accounting. The guard prevents double-counting in shared memory contexts.

### Two-Phase Teardown
`FreeTree()` calls **both** `FreeTreePortals_r()` **then** `FreeTree_r()` explicitly. This two-phase deletion is critical:
1. **Phase 1** (`FreeTreePortals_r`): Disconnect all portal links (critical for correctness—portals are doubly-linked across nodes)
2. **Phase 2** (`FreeTree_r`): Free all nodes and their content

Omitting Phase 1 would leave dangling portal pointers in freed memory.

## Data Flow Through This File

### Typical Flow (from compiler perspective)

1. **Construction**: q3map builds tree bottom-up via plane subdivision, creating `node_t` instances with `children[0/1]` pointers and portal links
2. **Traversal** (via `NodeForPoint`): Geometric queries during construction use `DotProduct()` classification to navigate from root toward appropriate leaf
3. **Serialization**: Tree is converted to `.bsp` file format (tree → lumps)
4. **Cleanup** (via `FreeTree`):
   - Walk tree recursively, unlinking portals first
   - Then recursively free all nodes, brushlists, and volume brushes
5. **Accounting**: `c_nodes` counter decremented to track peak memory use

### State Invariants

- **Before cleanup**: Tree is a complete binary DAG; every non-leaf has exactly 2 children; portals form bidirectional links
- **During cleanup**: Post-order ensures all children are freed before parent; dangling pointers are never dereferenced
- **After cleanup**: All nodes, portals, and brushes are returned to heap; `c_nodes` reflects freed count (single-threaded only)

## Learning Notes

### BSP Tree Concepts
- **Leaf node** identification via `planenum == PLANENUM_LEAF` (not a pointer-based type tag—pure sentinel value)
- **Plane-based partitioning**: Each non-leaf node divides space via a plane; child traversal uses signed distance test
- **Portal connectivity**: Portals link adjacent leaf regions for visibility/PVS computation (separate from tree structure itself)

### Era-Specific Patterns (Q3A, ~2005)
1. **C-style tree management**: No RAII; explicit two-phase teardown required. Modern engines use destructors or garbage collection
2. **Global state** (`c_nodes`, `mapplanes[]`, `numthreads`): Threading awareness baked into cleanup, reflecting late-1990s multicore porting
3. **Recursive descent** with no tail-call optimization guards. Real BSP trees can be deep; stack overflow possible on degenerate maps
4. **Debug printing in production code** (`PrintTree_r`): Reflects era when debug output was inline, not behind logging frameworks

### Comparison to Modern Engines
- Modern engines (Unreal, Unity, Godot) use **scene graphs** instead of BSP, with ECS/component models
- Spatial partitioning in modern engines is **rebuilt per-frame** (octrees, BVH) rather than precomputed offline
- Portal-based visibility is largely **obsolete**, replaced by GPU rasterization and compute shaders for culling

### Idiomatic to This Engine
The **offline/online split** is distinctive: q3map is a **tool pipeline** (build-time), separate from the runtime engine. This file is part of that pipeline—it will never execute during gameplay, only during level authoring. This justifies relatively simple memory management; performance is "good enough for build time."

## Potential Issues

### Lack of Null Checks
The code assumes `node` is never null in recursive functions. A corrupted tree (null pointer in a child slot) would cause immediate crash. The BSP construction code must guarantee tree integrity.

### Stack Depth
Deep recursion for unbalanced trees (linear chains of nodes) could overflow the stack. Q3A maps were typically balanced, so this was not a practical problem, but a degenerate map could trigger it.

### Portal Bidirectional Consistency
The reliance on `RemovePortalFromNode()` being called correctly means portal consistency is **not locally verifiable** in this file. If portals are inconsistently linked, the teardown could miss some cleanup or corrupt other structures.

---

**Cross-Reference Summary:** This file is a **utility layer** in the q3map compiler, sitting between the geometric tree-building phase and the serialization phase. It has no runtime role in the shipped engine (`code/` directory) and is purely a compile-time convenience.
