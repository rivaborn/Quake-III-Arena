# code/bspc/tree.c

## File Purpose
Manages the lifecycle and traversal of BSP trees used in the BSPC (BSP Compiler) tool. Provides allocation, deallocation, traversal, debug printing, and pruning of BSP tree nodes and their associated portals and brushes.

## Core Responsibilities
- Allocate and zero-initialize `tree_t` structures
- Recursively free portals attached to BSP nodes
- Recursively free brush lists, volume brushes, and node memory
- Traverse the BSP tree to find the leaf node containing a given 3D point
- Debug-print the BSP tree structure to stdout
- Prune redundant interior nodes where both children are solid (optimization pass)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `tree_t` | struct | Root BSP tree container holding headnode and world bounds |
| `node_t` | struct | BSP tree node/leaf; holds plane, children, brushes, portals, contents |
| `portal_t` | struct | Connectivity portal between two nodes; holds winding and linked lists |
| `bspbrush_t` | struct | Brush fragment stored at a leaf node; linked list |
| `plane_t` | struct | Map plane (normal + dist) used for spatial partitioning |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `c_nodes` | `int` (extern) | global | Running count of allocated BSP nodes; decremented on free |
| `c_pruned` | `int` | file-static (reset per call) | Count of pruned nodes in current prune pass |
| `freedtreemem` | `int` | file-static (reset per `Tree_Free`) | Accumulates bytes freed during tree deallocation (ME debug only) |

## Key Functions / Methods

### NodeForPoint
- **Signature:** `node_t *NodeForPoint(node_t *node, vec3_t origin)`
- **Purpose:** Descends the BSP tree from `node` to find the leaf containing `origin`.
- **Inputs:** Root/starting node, 3D point.
- **Outputs/Return:** Pointer to the leaf `node_t` (`planenum == PLANENUM_LEAF`).
- **Side effects:** None.
- **Calls:** `DotProduct` (macro).
- **Notes:** Terminates when `planenum == PLANENUM_LEAF`; uses plane sign of dot product to choose child.

### Tree_FreePortals_r
- **Signature:** `void Tree_FreePortals_r(node_t *node)`
- **Purpose:** Recursively frees all portals on every node in the subtree.
- **Inputs:** Subtree root node.
- **Outputs/Return:** void.
- **Side effects:** Frees portal windings and portal structs; sets `node->portals = NULL`; accumulates `freedtreemem` under `#ifdef ME`.
- **Calls:** `RemovePortalFromNode`, `FreePortal`, `MemorySize` (ME only).
- **Notes:** Must run before `Tree_Free_r` since portals reference nodes bidirectionally.

### Tree_Free_r
- **Signature:** `void Tree_Free_r(node_t *node)`
- **Purpose:** Recursively frees brush lists, volume brush, and the node itself for the entire subtree.
- **Inputs:** Subtree root node.
- **Outputs/Return:** void.
- **Side effects:** Decrements `c_nodes` (single-threaded only); frees memory; accumulates `freedtreemem`.
- **Calls:** `FreeBrush`, `FreeMemory`, `MemorySize` (ME only).
- **Notes:** Face freeing is commented out (Q2-only path). Volume brush freed separately from brushlist.

### Tree_Free
- **Signature:** `void Tree_Free(tree_t *tree)`
- **Purpose:** Entry point to fully deallocate a BSP tree (portals, nodes, tree struct).
- **Inputs:** Pointer to tree; no-op if NULL.
- **Outputs/Return:** void.
- **Side effects:** Calls `Tree_FreePortals_r` then `Tree_Free_r`; logs freed bytes under `#ifdef ME`.
- **Calls:** `Tree_FreePortals_r`, `Tree_Free_r`, `FreeMemory`, `Log_Print`, `PrintMemorySize`.

### Tree_Alloc
- **Signature:** `tree_t *Tree_Alloc(void)`
- **Purpose:** Allocates and zero-initializes a new `tree_t`.
- **Inputs:** None.
- **Outputs/Return:** Heap-allocated `tree_t*` with cleared bounds.
- **Side effects:** Heap allocation via `GetMemory`.
- **Calls:** `GetMemory`, `memset`, `ClearBounds`.

### Tree_PruneNodes_r
- **Signature:** `void Tree_PruneNodes_r(node_t *node)`
- **Purpose:** Recursively collapses interior nodes where both children are `CONTENTS_SOLID` into a single solid leaf, reducing tree complexity.
- **Inputs:** Subtree root node.
- **Outputs/Return:** void.
- **Side effects:** Modifies `node->planenum`, `node->contents`; frees child node memory; increments `c_pruned`; merges brush lists.
- **Calls:** `FreeMemory`, `Error`.
- **Notes:** Skips pruning if either child has `CONTENTS_LADDER` when `create_aas` is set (AAS pathfinding requires ladder nodes). Asserts no faces exist on solid-solid splits.

### Tree_PruneNodes
- **Signature:** `void Tree_PruneNodes(node_t *node)`
- **Purpose:** Logging wrapper that resets `c_pruned`, runs `Tree_PruneNodes_r`, and reports result.
- **Calls:** `Log_Print`, `Tree_PruneNodes_r`.

### Tree_Print_r
- **Signature:** `void Tree_Print_r(node_t *node, int depth)`
- **Purpose:** Debug-prints the BSP tree structure with indentation to stdout.
- **Calls:** `printf`.
- **Notes:** Trivial diagnostic utility; not used in production pipeline.

## Control Flow Notes
This file is part of the BSPC offline map-compilation tool (not the runtime engine). It operates during the BSP build pipeline: `Tree_Alloc` creates the tree, spatial subdivision populates it, `Tree_PruneNodes` optimizes it post-build, and `Tree_Free` tears it down. `NodeForPoint` is a utility used by flood-fill and leak-detection passes.

## External Dependencies
- **`qbsp.h`** — defines all core BSP types (`tree_t`, `node_t`, `portal_t`, `bspbrush_t`, `plane_t`), constants (`PLANENUM_LEAF`, `CONTENTS_SOLID`, `CONTENTS_LADDER`), and global arrays (`mapplanes`).
- **Defined elsewhere:** `RemovePortalFromNode` (portals.c), `FreePortal` (portals.c), `FreeBrush`/`FreeBrushList` (brushbsp.c), `GetMemory`/`FreeMemory`/`MemorySize` (l_mem.c), `ClearBounds` (l_math/l_cmd), `Log_Print`/`PrintMemorySize` (l_log.c), `numthreads` (l_threads.c), `create_aas` (bspc.c).
