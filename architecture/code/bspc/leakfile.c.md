# code/bspc/leakfile.c

## File Purpose
Generates a `.lin` leak trace file for the BSPC (BSP Compiler) tool. It traces the shortest portal path from the outside leaf to an occupied (entity-containing) leaf, enabling map editors like QE3 to visualize map leaks.

## Core Responsibilities
- Check whether the BSP tree's outside node is occupied (i.e., a leak exists)
- Traverse the portal graph greedily, following the path of decreasing `occupied` values
- Compute the center point of each portal winding along the path
- Write all trace points as XYZ coordinates to a `.lin` text file
- Append the final occupant entity's origin as the last point

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `tree_t` | struct (typedef) | Root BSP tree; holds `headnode` and `outside_node` |
| `node_t` | struct (typedef) | BSP node/leaf; carries `occupied`, `occupant`, and `portals` linkage |
| `portal_t` | struct (typedef) | BSP portal connecting two nodes; holds `winding`, `nodes[2]`, `next[2]` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `source` | `char[1024]` | global (defined in `bspc.c`) | Base filename used to construct the `.lin` output path |

## Key Functions / Methods

### LeakFile
- **Signature:** `void LeakFile(tree_t *tree)`
- **Purpose:** Traces and writes the shortest leak path through the BSP portal graph from the outside node to the nearest occupied leaf.
- **Inputs:** `tree` — pointer to the fully built and flood-filled BSP tree
- **Outputs/Return:** `void`; side effect is a written `.lin` file on disk
- **Side effects:** Opens and writes to `<source>.lin` via `fopen`/`fprintf`/`fclose`; prints diagnostics via `qprintf`; calls `Error` (fatal) if the file cannot be opened
- **Calls:**
  - `qprintf` — progress/diagnostic output
  - `sprintf` — constructs output filename
  - `fopen`, `fprintf`, `fclose` — file I/O
  - `Error` — fatal error on file-open failure
  - `WindingCenter` — computes the centroid of a portal winding
  - `GetVectorForKey` — retrieves the `"origin"` key from the occupant entity
- **Notes:**
  - Early-exit guard: returns immediately if `tree->outside_node.occupied == 0` (no leak)
  - The greedy traversal follows the portal whose destination node has the smallest `occupied` value less than the current node's — this is valid because `FloodEntities` (portals.c) assigns monotonically increasing `occupied` distances during flood fill
  - The loop terminates when `node->occupied == 1` (the leaf directly containing an entity); the entity origin is then appended as the final point
  - `s = (p->nodes[0] == node)` is the standard Q3 idiom for determining which side of a portal the current node is on; `p->next[!s]` walks the linked list for that node's portals
  - The `nextportal`/`nextnode` locals are only valid after the inner `for` loop finds at least one candidate — no null-check guard is present; correctness depends on the flood-fill invariant that a reachable path always exists

## Control Flow Notes
`LeakFile` is called during the BSP compilation pipeline after `FloodEntities` has marked reachable nodes. It is a diagnostic/output step: if the map leaks, compilation typically halts or warns, and this function produces the artifact that lets the level editor display the leak line. It has no role in the render or game-runtime loops.

## External Dependencies
- `qbsp.h` — aggregates all local BSPC headers; provides `tree_t`, `node_t`, `portal_t`, `winding_t`, entity types, and global declarations
- `WindingCenter` — defined in `l_poly.c` (via `l_poly.h`)
- `GetVectorForKey` — defined in `l_bsp_ent.c`
- `qprintf`, `Error` — defined in `l_cmd.c`
- `source` (global char array) — defined in `bspc.c`
