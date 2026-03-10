# code/bspc/aas_prunenodes.c — Enhanced Analysis

## Architectural Role

This file implements a critical optimization pass in the **offline BSPC (BSP-to-AAS compiler) pipeline**. It sits between the area-merging phase (`aas_areamerging.c`) and file serialization (`aas_file.c`), eliminating redundant internal BSP nodes after areas have been merged. The pruner reduces tree complexity and final AAS file size by collapsing nodes whose both children resolve to the same canonical area and removing purely-solid subtrees—essential cleanup before AAS serialization for bot runtime use.

## Key Cross-References

### Incoming (who depends on this file)
- **BSPC main compilation pipeline** (called from `be_aas_bspc.c` / `AAS_CalcReachAndClusters` context): Entry point `AAS_PruneNodes()` is invoked after areas are merged and before final AAS tree storage
- **Implied callers**: The offline map compiler must invoke this to optimize the AAS tree before writing

### Outgoing (what this file depends on)
- **`AAS_FreeTmpNode`** from `code/bspc/aas_create.c` — deallocates temporary BSP node structures
- **`tmpaasworld` global** from `code/bspc/aas_create.c` — holds the root node (`tmpaasworld.nodes`) and temporary AAS data
- **Logging layer** (`Log_Write`, `Log_Print`) — status and metrics reporting
- **Type definitions** from `aas_create.h` / `qbsp.h` — `tmp_node_t`, `tmp_area_t`

## Design Patterns & Rationale

### Post-Order Tree Traversal with Merge Chain Following
The recursive pruner uses a **post-order depth-first traversal** (process children first, then parent), which is idiomatic for bottom-up tree simplification. After recursively pruning children, it:

1. **Follows merge chains** (`while(tmparea->mergedarea)`) to resolve canonical areas — reflects that the area-merging phase created a *singly-linked chain* of merged areas, not a union-find structure
2. **Collapses redundant parents** when both children reference the same canonical area — a single area node replaces the internal split plane
3. **Frees orphaned solid subtrees** (two solid leaves) — cleanup of impossible branches

**Why this design?** 
- Late-stage tree simplification (after merging) avoids rebuilding the BSP; cheaper to collapse nodes than re-partition
- Preserves the BSP structure for prior passes while deferring optimization
- Post-order traversal ensures children are processed before deciding parent fate—classic bottom-up optimization pattern

## Data Flow Through This File

```
Input:  tmpaasworld.nodes (root of temporary BSP tree post-area-merge)
        ↓
        AAS_PruneNodes()
        ├─→ Log_Write("AAS_PruneNodes\r\n")
        ├─→ AAS_PruneNodes_r(tmpaasworld.nodes)  [recursive traversal]
        │   ├─→ For each internal node:
        │   │   ├─ Process children recursively
        │   │   ├─ If both children → same canonical area: collapse & free children
        │   │   ├─ If both children → NULL (solid): free parent, return NULL
        │   │   └─ Else: return node (possibly with pruned children)
        │   └─→ Updates tmpaasworld.nodes in-place
        └─→ Log_Print("%6d nodes pruned\r\n", c_numprunes)

Output: Modified tmpaasworld with pruned tree; count logged
        ↓
        [Next: AAS_StoreFile() serializes the optimized tree to disk]
```

## Learning Notes

### What This Teaches About Quake III's Architecture

1. **Offline-First Design**: AAS optimization is entirely offline; the runtime botlib (`code/botlib/`) never prunes—it loads pre-optimized `.aas` files. This separation of compilation from runtime is a classic engine pattern.

2. **Temporary vs. Persistent Structures**: BSPC operates on volatile `tmp_*` types (`tmp_node_t`, `tmp_area_t`) that exist only during compilation; the serialized AAS file uses a different, space-optimized format. Modern engines often collapse these phases.

3. **Merge Chain Pattern** (not union-find): The `mergedarea` linked list is a simple merge strategy. A modern implementation might use **union-find with path compression** or merge all areas into a single canonical list at once; the chain-following loop (`while(tmparea->mergedarea)`) is O(chain length) per query and could be O(n) in pathological cases.

4. **No CSE or Memoization**: The pruner recomputes canonical areas independently for each node's children; no caching of `find(area)` results. For large trees with deep merge chains, this could be optimized with memoization.

5. **Single-Pass Offline Preprocessing**: Unlike runtime optimization passes, this is one-shot and offline, so performance is less critical than correctness and file size reduction.

### Contrasts with Modern Engines

- **Modern approaches** (UE5, Unity) often **avoid building redundant geometry** in the first place (e.g., through hierarchical spatial subdivision that never creates unnecessary splits)
- **Modern AAS/navmesh systems** (Recast, Detour) use **single-pass generation** with post-processing filters rather than build-prune-serialize pipelines
- **Modern merge strategies** use **union-find or persistent data structures** for O(α(n)) or O(log n) amortized cost instead of traversing merge chains

## Potential Issues

### Potential Inefficiency: Repeated Merge Chain Traversal
For each internal node with area children, the code traverses merge chains independently:
```c
while(tmparea1->mergedarea) tmparea1 = tmparea1->mergedarea;
while(tmparea2->mergedarea) tmparea2 = tmparea2->mergedarea;
```
In a tree with many nodes and long merge chains, this is **O(nodes × chain_depth)**. If many areas were merged, this could be slow. A single pass to memoize canonical areas before pruning would be O(areas + nodes).

### Silent Correctness Assumption: Tree Structure Validity
The pruner assumes:
- `AAS_FreeTmpNode` correctly deallocates children (no double-free bugs in callee)
- Merge chains are acyclic and terminate (no infinite loops)
- Area pointers in nodes are consistent with global `tmpaasworld`

If a prior pass (area merging) corrupts the tree or merge chains, the pruner will silently propagate the error.

---

**Summary**: This file is a **textbook tree optimization pass** in an offline compiler pipeline, using idiomatic post-order traversal and simple merge-chain following. It reveals Quake III's strategy of deferring geometric simplification to a dedicated compilation phase, in contrast to modern engines that front-load complexity reduction.
