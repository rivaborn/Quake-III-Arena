# code/bspc/prtfile.c

## File Purpose
Generates the `.prt` (portal file) used by the `qvis` visibility compiler. It traverses the BSP tree to enumerate vis-clusters and portals between them, writes the `PRT1`-format file, and stores cluster assignments back into the BSP leaf array.

## Core Responsibilities
- Recursively traverse the BSP tree to assign cluster numbers to leaf nodes
- Count vis-clusters and vis-portals for the portal file header
- Write portal geometry (winding points + cluster pair indices) to `name.prt`
- Handle detail-separator nodes as cluster boundaries (collapsing subtrees into one cluster)
- Rebuild portals suited for vis before writing (free old portals, create head-node portals, split)
- Propagate final cluster IDs back into `dleafs[]` after BSP write ordering

## Key Types / Data Structures
None defined here; uses types from `qbsp.h`.

| Name | Kind | Purpose |
|---|---|---|
| `node_t` | struct (extern) | BSP tree node/leaf; holds cluster, portals, contents |
| `portal_t` | struct (extern) | Portal between two nodes; has winding and plane |
| `winding_t` | struct (extern) | Convex polygon; array of `vec3_t` points |
| `tree_t` | struct (extern) | BSP tree root with headnode |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `pf` | `FILE *` | global | Open handle to the `.prt` output file |
| `num_visclusters` | `int` | global | Running count of player-reachable vis clusters |
| `num_visportals` | `int` | global | Running count of flood-visible portals |
| `clusterleaf` | `int` | global | Index cursor for `SaveClusters_r` leaf iteration |

## Key Functions / Methods

### WriteFloat2
- **Signature:** `void WriteFloat2(FILE *f, vec_t v)`
- **Purpose:** Writes a float to file as integer if near-integer, else as `%f`, minimizing file size.
- **Inputs:** File pointer, float value
- **Outputs/Return:** None (writes to file)
- **Side effects:** `fprintf` I/O to `f`
- **Calls:** `fabs`, `Q_rint`, `fprintf`

### WritePortalFile_r
- **Signature:** `void WritePortalFile_r(node_t *node)`
- **Purpose:** Recursively visits every vis-leaf and emits one line per outbound flood-visible portal: `numpoints clusterA clusterB (x y z) ...`
- **Inputs:** BSP node
- **Outputs/Return:** None
- **Side effects:** Writes to global `pf`; reads `node->portals`, `portal_t` fields
- **Calls:** `WindingPlane`, `DotProduct`, `WriteFloat2`, `Portal_VisFlood`, `fprintf`
- **Notes:** Stops recursion at `detail_seperator` nodes and solid leaves. Checks plane orientation vs. computed winding plane and swaps cluster order if mismatched (backward-plane guard).

### FillLeafNumbers_r
- **Signature:** `void FillLeafNumbers_r(node_t *node, int num)`
- **Purpose:** Stamps every node/leaf in a detail subtree with the same cluster number (collapsing the detail subtree into one cluster).
- **Inputs:** Node, cluster number to assign
- **Side effects:** Mutates `node->cluster` throughout subtree
- **Calls:** Recursive self

### NumberLeafs_r
- **Signature:** `void NumberLeafs_r(node_t *node)`
- **Purpose:** DFS traversal; assigns a unique cluster number to each non-solid vis-leaf (or detail-separator subtree), increments `num_visclusters`, counts flood-visible portals into `num_visportals`.
- **Inputs:** BSP node
- **Side effects:** Mutates `node->cluster`; increments `num_visclusters`, `num_visportals`
- **Calls:** `FillLeafNumbers_r`, `Portal_VisFlood`

### CreateVisPortals_r
- **Signature:** `void CreateVisPortals_r(node_t *node)`
- **Purpose:** Recursively builds node portals down to detail-separator boundaries; below a detail separator everything is one cluster so no further splitting is needed.
- **Calls:** `MakeNodePortal`, `SplitNodePortals`

### FinishVisPortals_r / FinishVisPortals2_r
- **Signature:** `void FinishVisPortals_r(node_t *node)` / `void FinishVisPortals2_r(node_t *node)`
- **Purpose:** Completes portal construction inside detail-separator subtrees (used as a post-pass). `FinishVisPortals_r` skips non-detail nodes; `FinishVisPortals2_r` handles the interior.
- **Calls:** `MakeNodePortal`, `SplitNodePortals`

### SaveClusters_r
- **Signature:** `void SaveClusters_r(node_t *node)`
- **Purpose:** Stores each leaf's cluster ID into the `dleafs[]` BSP output array using `clusterleaf` as a cursor.
- **Side effects:** Writes `dleafs[clusterleaf++].cluster`; increments global `clusterleaf`

### WritePortalFile
- **Signature:** `void WritePortalFile(tree_t *tree)`
- **Purpose:** Top-level entry point. Rebuilds portals, numbers clusters, writes the `.prt` file, then saves cluster assignments to `dleafs[]`.
- **Inputs:** Compiled BSP tree
- **Outputs/Return:** None
- **Side effects:** File creation (`name.prt`); mutates `dleafs[]`; resets `num_visclusters`, `num_visportals`, `clusterleaf`
- **Calls:** `Tree_FreePortals_r`, `MakeHeadnodePortals`, `CreateVisPortals_r`, `NumberLeafs_r`, `sprintf`, `fopen`, `fprintf`, `WritePortalFile_r`, `fclose`, `SaveClusters_r`, `Error`, `qprintf`

## Control Flow Notes
Called during the BSP compilation pipeline (from `bspc.c`) after the BSP is structurally complete but before or alongside `WriteBSPFile`. `WritePortalFile` is a one-shot, offline-tool pass: init → portal rebuild → cluster numbering → file write → cluster save. No frame/update loop involvement.

## External Dependencies
- **`qbsp.h`** — all core types (`node_t`, `portal_t`, `winding_t`, `tree_t`, `plane_t`) and declarations
- **`source`** (extern `char[1024]`, defined in `bspc.c`) — base filename for output path
- **`dleafs[]`** (defined in BSP file I/O layer, e.g. `l_bsp_q2.c`/`aas_file.c`) — output BSP leaf array
- **`Portal_VisFlood`**, **`MakeNodePortal`**, **`SplitNodePortals`**, **`MakeHeadnodePortals`**, **`Tree_FreePortals_r`** — defined in `portals.c` / `tree.c`
- **`WindingPlane`**, **`DotProduct`**, **`Q_rint`** — math utilities from `l_math.c` / `l_poly.c`
- **`Error`**, **`qprintf`** — logging/error utilities from `l_cmd.c`
