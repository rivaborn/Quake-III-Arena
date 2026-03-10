# code/bspc/portals.c

## File Purpose
Implements BSP portal generation, entity flood-fill, and area classification for the BSPC (BSP Compiler) tool. Portals are convex polygon boundaries between adjacent BSP leaf nodes, used for PVS (Potentially Visible Set) computation and area portal detection.

## Core Responsibilities
- Allocate/free `portal_t` objects with memory tracking
- Build axis-aligned bounding portals for the BSP headnode (`MakeHeadnodePortals`)
- Create and split node portals during BSP tree traversal
- Flood-fill the BSP tree from entity origins to identify reachable vs. outside space
- Fill unreachable leaves as solid (`FillOutside`)
- Flood-classify leaves into numbered areas separated by `CONTENTS_AREAPORTAL` nodes
- Mark brush sides as visible when referenced by portals

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `portal_t` | struct (defined in qbsp.h) | Convex polygon boundary between two BSP nodes; carries plane, winding, side reference |
| `node_t` | struct (defined in qbsp.h) | BSP tree node or leaf; holds portal list, contents, occupation state, area ID |
| `tree_t` | struct (defined in qbsp.h) | Root of the BSP tree with headnode and bounding box |
| `plane_t` | struct (defined in qbsp.h) | Normal + distance representation of a half-space |
| `winding_t` | struct (from l_poly) | Ordered vertex list representing a convex polygon |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `c_active_portals` | `int` | global | Count of currently live portals (single-thread only) |
| `c_peak_portals` | `int` | global | High-water mark of simultaneous portals |
| `c_boundary` | `int` | global | Boundary portal counter (unused in visible code) |
| `c_boundary_sides` | `int` | global | Boundary side counter (unused in visible code) |
| `c_portalmemory` | `int` | global | Running byte total of portal allocations |
| `c_tinyportals` | `int` | global | Count of degenerate portals discarded |
| `c_numportalizednodes` | `int` | global | Progress counter for `MakeTreePortals_r` |
| `p_firstnode` / `p_lastnode` | `node_t *` | global | Head/tail of BFS queue used by `FloodPortals` |
| `c_outside` / `c_inside` / `c_solid` | `int` | global | Leaf classification counts for `FillOutside` |
| `c_areas` | `int` | global | Running area ID counter for `FloodAreas` |
| `numrec` | `int` | global | Recursion depth counter for `FloodPortals_r` |

## Key Functions / Methods

### AllocPortal
- **Signature:** `portal_t *AllocPortal(void)`
- **Purpose:** Allocates and zero-initializes a new portal; tracks count and memory in single-thread mode.
- **Inputs:** None
- **Outputs/Return:** Pointer to new `portal_t`
- **Side effects:** Increments `c_active_portals`, updates `c_peak_portals`, adds to `c_portalmemory`
- **Calls:** `GetMemory`, `memset`, `MemorySize`

### FreePortal
- **Signature:** `void FreePortal(portal_t *p)`
- **Purpose:** Frees portal's winding then the portal itself; decrements tracking counters.
- **Side effects:** Decrements `c_active_portals`, subtracts from `c_portalmemory`
- **Calls:** `FreeWinding`, `MemorySize`, `FreeMemory`

### VisibleContents
- **Signature:** `int VisibleContents(int contents)`
- **Purpose:** Returns the single highest-priority visible content bit from a combined content mask.
- **Inputs:** Bitfield of content flags
- **Outputs/Return:** Single content bit, or 0 if none visible

### ClusterContents
- **Signature:** `int ClusterContents(node_t *node)`
- **Purpose:** Recursively ORs all leaf contents under a subtree; strips `CONTENTS_SOLID` when only one child has it (partial solid clusters remain visible).
- **Calls:** Itself recursively

### Portal_VisFlood
- **Signature:** `qboolean Portal_VisFlood(portal_t *p)`
- **Purpose:** Determines if the PVS can see through this portal. Returns false for unlinked or solid-blocked portals.
- **Calls:** `ClusterContents`, `VisibleContents`

### Portal_EntityFlood
- **Signature:** `qboolean Portal_EntityFlood(portal_t *p, int s)`
- **Purpose:** Returns true if entities can flood through this portal (non-solid leaf on both sides). Used for entity reachability and area classification.
- **Notes:** Both nodes must be leaves (`PLANENUM_LEAF`) or an error is raised.

### AddPortalToNodes / RemovePortalFromNode
- **Purpose:** Link/unlink a portal into the singly-linked portal lists of its two adjacent nodes via the `next[2]` side-indexed linked list.
- **Notes:** `RemovePortalFromNode` includes a circular-link debug check over a local stack of 4096 portals.

### MakeHeadnodePortals
- **Signature:** `void MakeHeadnodePortals(tree_t *tree)`
- **Purpose:** Creates 6 axis-aligned bounding portals around the tree's AABB (padded by `SIDESPACE=8`), connects them to `outside_node`, then clips each portal against the other 5.
- **Calls:** `AllocPortal`, `BaseWindingForPlane`, `AddPortalToNodes`, `ChopWindingInPlace`

### BaseWindingForNode
- **Signature:** `winding_t *BaseWindingForNode(node_t *node)`
- **Purpose:** Generates a large winding from the node's splitting plane, then clips it by all ancestor planes to produce the maximal portal polygon at that split.
- **Calls:** `BaseWindingForPlane`, `ChopWindingInPlace`, `VectorSubtract`

### MakeNodePortal
- **Signature:** `void MakeNodePortal(node_t *node)`
- **Purpose:** Creates a single portal at the node's splitting plane, clips it against existing portals on the node, and connects it between the two children.
- **Calls:** `BaseWindingForNode`, `ChopWindingInPlace`, `WindingIsTiny`, `AllocPortal`, `AddPortalToNodes`

### SplitNodePortals
- **Signature:** `void SplitNodePortals(node_t *node)`
- **Purpose:** For each portal on a node being subdivided, clips it against the node's plane and reassigns the resulting front/back fragments to the child nodes.
- **Calls:** `RemovePortalFromNode`, `ClipWindingEpsilon`, `WindingIsTiny`, `AllocPortal`, `AddPortalToNodes`, `FreeWinding`

### MakeTreePortals_r / MakeTreePortals
- **Purpose:** Recursive DFS entry point; calls `CalcNodeBounds`, `MakeNodePortal`, `SplitNodePortals` at each internal node, then recurses into children.
- **Calls:** `CalcNodeBounds`, `MakeNodePortal`, `SplitNodePortals`, itself

### FloodPortals / PlaceOccupant / FloodEntities
- **Purpose:** BFS flood from entity origins through non-solid portals to mark all reachable nodes as `occupied`. `FloodEntities` iterates all non-world entities; `info_player_start` is nudged ±16 units if initial placement fails.
- **Side effects:** Sets `node->occupied` and `node->occupant`; uses `p_firstnode`/`p_lastnode` queue globals

### FillOutside_r / FillOutside
- **Purpose:** Recursively marks any unoccupied non-solid leaf as solid (`CONTENTS_SOLID`), effectively filling void space unreachable by entities.
- **Side effects:** Modifies `node->contents`; updates `c_outside`, `c_inside`, `c_solid`

### FloodAreas_r / FindAreas_r / SetAreaPortalAreas_r / FloodAreas
- **Purpose:** Assigns integer area IDs (`c_areas`) to reachable, non-solid leaves; stops at `CONTENTS_AREAPORTAL` nodes and records which two areas each area portal bridges.

### FindPortalSide / MarkVisibleSides_r / MarkVisibleSides
- **Purpose:** For each portal touching a non-empty leaf, finds the best-matching brush side (by plane alignment) and sets `SFL_VISIBLE` on it.
- **Calls:** `VisibleContents`, `DotProduct`, `Log_Print`

## Control Flow Notes
- Called from the main BSP pipeline: after `BrushBSP` builds the tree, `MakeTreePortals` portalize it, then `FloodEntities`→`FillOutside`→`FloodAreas`→`MarkVisibleSides` run in sequence to prepare the tree for AAS generation and BSP output.
- `MakeTreePortals_r` is a DFS post-order traversal; `FloodPortals` uses an iterative BFS queue.

## External Dependencies
- **`qbsp.h`**: All core BSP types (`portal_t`, `node_t`, `tree_t`, `plane_t`, `side_t`, etc.), map globals (`mapplanes`, `nummapplanes`, `entities`, `num_entities`), and winding utilities
- **`l_mem.h`**: `GetMemory`, `FreeMemory`, `MemorySize`
- **Defined elsewhere**: `BaseWindingForPlane`, `ChopWindingInPlace`, `ClipWindingEpsilon`, `FreeWinding`, `WindingIsTiny`, `WindingMemory`, `BaseWindingForNode` (winding ops from `l_poly`); `Log_Print`, `Log_Write` (logging); `Error`, `qprintf` (utility); `GetVectorForKey`, `ValueForKey` (entity key access); `numthreads`, `cancelconversion` (global compile-session flags); `DotProduct`, `VectorSubtract`, `VectorCopy`, `VectorCompare`, `ClearBounds`, `AddPointToBounds` (math macros/functions)
