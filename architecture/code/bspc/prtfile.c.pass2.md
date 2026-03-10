# code/bspc/prtfile.c — Enhanced Analysis

## Architectural Role

This file bridges the offline **BSP compilation pipeline** (within `bspc`, the standalone map processing tool) and the **`qvis` visibility compiler**. It executes a critical mid-pipeline step: after the BSP tree is structurally complete, `prtfile.c` extracts portal geometry and spatial cluster assignments from the tree, then writes them to the `PRT1` format file that `qvis` consumes to compute per-cluster visibility (PVS/PHS). The cluster assignments are subsequently copied back into the `dleafs[]` array so the final BSP incorporates visibility-relevant metadata.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** — calls `WritePortalFile(tree)` once per map compilation, as part of the main offline pipeline
- **Global `source[]`** (`char[1024]`, defined in `bspc.c`) — provides the base filename for output (e.g., `"maps/q3dm1"` → `"maps/q3dm1.prt"`)
- **Global `dleafs[]`** — BSP leaf array (from `code/bspc/l_bsp_q3.c` or equivalent), receives final cluster IDs via `SaveClusters_r()`

### Outgoing (what this file depends on)
- **Portal management** (`code/bspc/portals.c` or internal tree module):
  - `Tree_FreePortals_r()` — clears stale portals before rebuild
  - `MakeHeadnodePortals()` — initializes portals at tree root
  - `CreateVisPortals_r()` / `FinishVisPortals_r()` — recursively constructs portals down the tree
  - `Portal_VisFlood()` — tests whether a portal is "flood-visible" (participates in cluster connectivity)
- **Math utilities** (from `code/bspc/l_math.c` / `code/bspc/l_poly.c`):
  - `WindingPlane()`, `DotProduct()`, `Q_rint()` — geometry calculations
- **Logging** (`code/bspc/l_cmd.c`):
  - `Error()`, `qprintf()` — error reporting and progress output
- **Standard I/O**: `fopen()`, `fprintf()`, `fclose()`, `sprintf()` — file operations

## Design Patterns & Rationale

**Offline-Tool Architecture:**
- **Single-pass, one-shot operation**: No frame loop; called once per map compilation as part of the tool pipeline.
- **Procedural tree traversal**: Recursive DFS with global accumulation (counters, file handle), which is idiomatic for offline tools but would be a code smell in runtime engine code.
- **Deferred file write**: All computation (`NumberLeafs_r`, portal enumeration) completes before atomic file creation, minimizing I/O overhead and corruption risk.
- **Post-hoc state propagation**: Cluster IDs are computed and written to `.prt`, then separately copied back into `dleafs[]` via `SaveClusters_r()` — this ordering avoids coupling the vis-numbering logic to BSP file output.

**Detail-Separator Optimization:**
- The code checks `node->detail_seperator` to stop portal recursion early. Detail geometry (e.g., clipped brushes, decoration) is collapsed into a single cluster below a detail boundary, so `qvis` never traverses below it.
- This is a Quake III-specific technique for handling huge maps without combinatorial explosion in PVS computation.

**Plane Orientation Guard** (in `WritePortalFile_r`):
- Compares `DotProduct(p->plane.normal, computed_winding_normal)` and swaps cluster order if the dot product is < 0.99. This handles cases where portal planes may flip during tree construction, ensuring `qvis` reads consistent winding orientation.
- The `// FIXME: is this still relevant?` comment suggests this may be legacy defensive code.

## Data Flow Through This File

1. **Input State**:
   - `node_t` BSP tree (complete but portals not yet vis-suitable)
   - `tree->headnode` as entry point

2. **Initialization** (in `WritePortalFile`):
   - Clear old portals via `Tree_FreePortals_r()`
   - Rebuild portals with `MakeHeadnodePortals()` + `CreateVisPortals_r()` + optional `FinishVisPortals_r()` for detail subtrees

3. **Cluster Assignment** (via `NumberLeafs_r()`):
   - DFS each node; for each non-solid leaf (or detail-separator subtree), call `FillLeafNumbers_r()` to stamp a unique cluster ID, then increment `num_visclusters`
   - Count flood-visible portals into `num_visportals`

4. **File Output** (via `WritePortalFile_r()`):
   - Open `"<source>.prt"` for writing
   - Write header: `"PRT1"`, `num_visclusters`, `num_visportals`
   - Recurse tree; for each outbound flood-visible portal: write `numpoints clusterA clusterB (x₀ y₀ z₀) ...` (with optional plane-flip handling)

5. **Post-Processing** (via `SaveClusters_r()`):
   - Linear DFS through leaves; copy `node->cluster` into `dleafs[clusterleaf++].cluster`
   - This synchronizes the in-core BSP leaf array with computed clusters before BSP serialization

## Learning Notes

**Idiomatic to this era/engine:**
- **Global state accumulation** (`num_visclusters`, `num_visportals`, `clusterleaf`) — common in 1990s offline tools; trades encapsulation for simplicity.
- **Recursive tree traversal** — no iterative worklist or BFS queue; pre-dates functional/iterative styles.
- **No error recovery on file I/O** — assumes `fopen()` will succeed; on failure, `Error()` halts the tool. Acceptable for offline tools.
- **Detail-separator semantics** — Quake III-specific optimization; modern engines often use spatial acceleration (octrees, grids) or just disable PVS entirely.
- **Manual cluster numbering** — hardcoded in-order traversal; no formal cluster graph abstraction.

**Contrast with modern approaches:**
- Modern game engines often pre-compute a cluster connectivity graph explicitly, then serialize it; here, the cluster graph is implicit in the portal list.
- Modern tools might validate the output (e.g., check that all clusters are reachable) before writing; this code does not.
- Modern offline compilers often use persistent data structures or multi-pass algorithms; this is single-pass and destructive (portals are freed then recreated).

**Connection to Engine Concepts:**
- **Portal/cell/cluster architecture**: This file exemplifies the **portal/cell** visibility approach (as opposed to modern PVS rasterization or precomputed visibility octrees).
- **Spatial partitioning**: The BSP tree *is* the spatial partition; portals are the edges between cells.
- **Offline data baking**: Cluster assignments are baked into the BSP so the engine can look up cluster membership at O(1) runtime.

## Potential Issues

1. **Plane-Orientation Assumption** (line 82–88, `WritePortalFile_r`):
   - The FIXME comment suggests the backward-plane guard may be unnecessary legacy code. If this check is redundant, it wastes ~1 string comparison per portal.

2. **Unbounded Filename** (line 253):
   - `sprintf(filename, "%s.prt", source)` has no bounds check. If `source` exceeds ~1019 bytes (given a 1024-byte buffer), this will overflow. In practice, `source` comes from command-line args, so overflow is unlikely, but the code is technically unsafe.

3. **No Validation of Detail-Separator Placement**:
   - If the BSP tree structure is malformed (e.g., detail separator placed incorrectly), the cluster numbering may silently produce incorrect results. No asserts or checks validate tree structure.

4. **Portal Count May Be Incomplete**:
   - `num_visportals` counts only **flood-visible** portals; non-flood portals are silently discarded. If a portal fails `Portal_VisFlood()`, it won't appear in the `.prt` file. This is intentional but undocumented.

5. **Global File Handle Not Restored**:
   - `FILE *pf` remains open (or garbage) after `WritePortalFile()` completes. In a tool context, process exit cleans it up, but if the function were called multiple times, the handle would leak. (Minor issue given offline-tool context.)
