# code/bspc/aas_prunenodes.c

## File Purpose
This file implements a BSP tree pruning pass for the AAS (Area Awareness System) build tool (BSPC). It eliminates redundant internal BSP nodes where both children resolve to the same merged area, and collapses double-solid-leaf nodes, reducing AAS tree complexity before final file output.

## Core Responsibilities
- Recursively traverse the temporary AAS BSP node tree post-area-merge
- Detect and collapse internal nodes whose two children reference the same final area (after following merge chains)
- Detect and free double-solid-leaf nodes (both children NULL)
- Free redundant child nodes via `AAS_FreeTmpNode`
- Count and report the total number of pruned nodes

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `tmp_node_t` | struct | Temporary AAS BSP node; holds plane number, area pointer, and two child node pointers |
| `tmp_area_t` | struct | Temporary AAS area; includes a `mergedarea` chain pointer used to resolve the canonical area after merging |
| `tmp_aas_t` | struct | Global temporary AAS world; `tmpaasworld.nodes` is the root passed to the pruner |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `c_numprunes` | `int` | global | Running count of nodes pruned; reported at end of `AAS_PruneNodes` |

## Key Functions / Methods

### AAS_PruneNodes_r
- **Signature:** `tmp_node_t *AAS_PruneNodes_r(tmp_node_t *tmpnode)`
- **Purpose:** Recursive post-order BSP tree pruner. Collapses nodes that are redundant after area merging.
- **Inputs:** `tmpnode` — pointer to current `tmp_node_t`; NULL represents a solid leaf
- **Outputs/Return:** Pointer to the (possibly collapsed) node, or NULL if the subtree was pruned to a solid leaf
- **Side effects:** Frees child nodes via `AAS_FreeTmpNode`; increments `c_numprunes`; mutates `tmpnode->tmparea`, `tmpnode->planenum`, `tmpnode->children[0/1]`
- **Calls:** `AAS_PruneNodes_r` (recursive), `AAS_FreeTmpNode`
- **Notes:**
  - Solid leaf is represented by `tmpnode == NULL` (not a sentinel node)
  - Area-leaf is represented by `tmpnode->tmparea != NULL`
  - Merge chain is followed with `while(tmparea->mergedarea)` to get the canonical area; two children referencing the same canonical area collapse into a single area node
  - When both children are NULL (two solid leaves), the internal node itself is freed and NULL is returned

### AAS_PruneNodes
- **Signature:** `void AAS_PruneNodes(void)`
- **Purpose:** Entry point for the pruning pass; logs start, invokes the recursive traversal from the world root, and prints the prune count.
- **Inputs:** None (uses `tmpaasworld.nodes` as root)
- **Outputs/Return:** void
- **Side effects:** Writes to log via `Log_Write`/`Log_Print`; modifies tree in place via `AAS_PruneNodes_r`; updates `c_numprunes`
- **Calls:** `Log_Write`, `AAS_PruneNodes_r`, `Log_Print`
- **Notes:** No return value; caller must check `tmpaasworld.nodes` for the (possibly modified) root after the call.

## Control Flow Notes
This file is part of the BSPC offline map compilation pipeline. `AAS_PruneNodes` is called after area creation and merging (`aas_areamerging.c`) and before AAS file serialization (`aas_file.c`). It is a single-pass, offline preprocessing step with no frame or runtime involvement.

## External Dependencies
- `qbsp.h` — core BSPC types (`tmp_node_t`, etc. indirectly via `aas_create.h`), logging utilities
- `botlib/aasfile.h` — AAS file format constants and structures (included for type definitions)
- `aas_create.h` — defines `tmp_node_t`, `tmp_area_t`, `tmp_aas_t`, `tmpaasworld` global, and `AAS_FreeTmpNode`
- `Log_Write`, `Log_Print` — defined elsewhere in the BSPC logging layer (`l_log.c`)
- `AAS_FreeTmpNode` — defined in `aas_create.c`
- `tmpaasworld` — global `tmp_aas_t` instance defined in `aas_create.c`
