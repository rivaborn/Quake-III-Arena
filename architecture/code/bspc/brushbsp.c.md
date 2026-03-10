# code/bspc/brushbsp.c

## File Purpose
Implements the core BSP tree construction algorithm for the BSPC tool, converting a flat list of brushes into a binary space partitioning tree. It handles brush allocation/deallocation, plane-side testing, split-plane selection heuristics, brush splitting, and multithreaded iterative tree building.

## Core Responsibilities
- Allocate, copy, free, and bound `bspbrush_t` and `node_t` structures
- Determine which side of a plane a brush or AABB lies on (`BoxOnPlaneSide`, `TestBrushToPlanenum`)
- Select the best split plane from candidate brush sides using a cost heuristic (`SelectSplitSide`)
- Geometrically split a brush across a plane, producing two child brushes (`SplitBrush`)
- Partition a brush list into front/back children for a node (`SplitBrushList`)
- Build the BSP tree recursively (`BuildTree_r`) or iteratively with a thread-safe node queue/stack (`BuildTree`, `BuildTreeThread`)
- Classify leaf nodes with content flags and AAS-specific data (`LeafNode`)
- Track peak memory statistics for brushes, nodes, and windings

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bspbrush_t` | struct (defined in qbsp.h) | A brush fragment with sides, windings, bounds, and side-classification cache |
| `node_t` | struct (defined in qbsp.h) | BSP tree node or leaf; holds plane, children, brush list, volume brush, content flags |
| `side_t` | struct (defined in qbsp.h) | One face of a brush: plane number, texture info, winding, flags |
| `plane_t` | struct (defined in qbsp.h) | Map plane with normal, dist, type, and sign-bit cache for fast AABB tests |
| `tree_t` | struct (defined in qbsp.h) | Root container: head node and world bounds |
| `cname_t` | struct | Maps a content flag integer to a debug name string |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `c_nodes` | int | global | Total BSP nodes created |
| `c_nonvis` | int | global | Non-visible split nodes count |
| `c_active_brushes` | int | global | Currently allocated brush count |
| `c_solidleafnodes` | int | global | Solid leaf node count |
| `c_totalsides` | int | global | Total brush sides counted at BSP entry |
| `c_brushmemory` | int | global | Current brush memory usage (bytes) |
| `c_peak_brushmemory` | int | global | Peak brush memory usage |
| `c_nodememory` | int | global | Current node memory usage |
| `c_peak_totalbspmemory` | int | global | Peak combined BSP memory (nodes+brushes+windings) |
| `numrecurse` | int | global | Running count of nodes processed (progress display) |
| `firstnode` / `lastnode` | `node_t *` | global | Head/tail of the threaded node work list |
| `nodelistsize` | int | global | Current node list length |
| `use_nodequeue` | int | global | 0 = stack (depth-first), 1 = queue (breadth-first) |
| `numwaiting` | int | global | Threads currently blocked waiting for node list |
| `AddNodeToList` | function pointer | global | Dispatches to `AddNodeToStack` or `AddNodeToQueue` |
| `contentnames[]` | `cname_t[]` | file-static (no `static` keyword but file-local array) | Content flag → name table for debug printing |

## Key Functions / Methods

### ResetBrushBSP
- **Signature:** `void ResetBrushBSP(void)`
- **Purpose:** Zeroes all global BSP statistics counters before a new compile pass.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes 9 global counters.
- **Calls:** None
- **Notes:** Must be called before `BrushBSP` if reusing the tool for multiple maps.

### AllocBrush / FreeBrush / AllocNode
- **Notes:** `AllocBrush(numsides)` allocates a variably-sized `bspbrush_t` via `GetMemory` and updates `c_active_brushes`/`c_brushmemory` (single-thread only). `FreeBrush` frees all side windings then the brush block. `AllocNode` zeroes a `node_t` and tracks `c_nodememory`.

### BoundBrush
- **Signature:** `void BoundBrush(bspbrush_t *brush)`
- **Purpose:** Recomputes `brush->mins/maxs` from all side windings.
- **Inputs:** `brush` — brush with populated side windings.
- **Outputs/Return:** Updates `brush->mins`, `brush->maxs` in place.
- **Side effects:** None beyond the brush struct.
- **Calls:** `ClearBounds`, `AddPointToBounds`

### CreateBrushWindings
- **Signature:** `void CreateBrushWindings(bspbrush_t *brush)`
- **Purpose:** For each side, clips a full-plane winding by all other (non-bevel) sides to produce the actual polygon.
- **Inputs:** `brush` with plane numbers set on all sides.
- **Outputs/Return:** Populates `side->winding`; calls `BoundBrush`.
- **Side effects:** Allocates winding memory.
- **Calls:** `BaseWindingForPlane`, `ChopWindingInPlace`, `BoundBrush`

### BoxOnPlaneSide
- **Signature:** `int BoxOnPlaneSide(vec3_t emins, vec3_t emaxs, plane_t *p)`
- **Purpose:** Fast AABB vs. plane classification using precomputed `signbits` to select the extreme vertices.
- **Inputs:** AABB bounds, plane.
- **Outputs/Return:** Bitmask of `PSIDE_FRONT | PSIDE_BACK`.
- **Side effects:** None.
- **Notes:** Handles axial planes as a special fast path; 8-case switch on `signbits` for general planes.

### TestBrushToPlanenum
- **Signature:** `int TestBrushToPlanenum(bspbrush_t *brush, int planenum, int *numsplits, qboolean *hintsplit, int *epsilonbrush)`
- **Purpose:** Precise per-winding-vertex test of a brush against a candidate split plane; counts how many visible sides are actually split.
- **Inputs:** Brush, plane index, out-params for split count, hint flag, epsilon count.
- **Outputs/Return:** `PSIDE_*` flags; populates `*numsplits`, `*hintsplit`, `*epsilonbrush`.
- **Side effects:** Sets `SFL_TESTED` on matching sides.
- **Calls:** `BoxOnPlaneSide`

### SelectSplitSide
- **Signature:** `side_t *SelectSplitSide(bspbrush_t *brushes, node_t *node)`
- **Purpose:** Heuristic search over all candidate sides to find the best BSP split plane. Scores each candidate as `5*facing - 5*splits - abs(front-back)`, with bonuses for axial planes and heavy penalties for epsilon brushes or splitting hint planes.
- **Inputs:** Current brush list, current node (for parent/volume checks).
- **Outputs/Return:** Best `side_t *`, or NULL if no valid split exists.
- **Side effects:** Temporarily sets `SFL_TESTED` flags (cleared before return); writes `brush->side`/`testside`.
- **Calls:** `CheckPlaneAgainstParents`, `CheckPlaneAgainstVolume`, `TestBrushToPlanenum`

### SplitBrush
- **Signature:** `void SplitBrush(bspbrush_t *brush, int planenum, bspbrush_t **front, bspbrush_t **back)`
- **Purpose:** Geometrically splits a brush across a plane into front and back fragments; adds a mid-winding cap face to each half.
- **Inputs:** Source brush, plane index.
- **Outputs/Return:** `*front` and `*back` (either may be NULL if brush is entirely on one side or result is degenerate).
- **Side effects:** Allocates new brushes and windings; may free degenerate results.
- **Calls:** `CopyBrush`, `BaseWindingForPlane`, `ChopWindingInPlace`, `BrushMostlyOnSide`, `WindingIsTiny`, `WindingIsHuge`, `AllocBrush`, `ClipWindingEpsilon`, `BoundBrush`, `BrushVolume`, `FreeBrush`, `CopyWinding`, `FreeWinding`, `Log_Write`

### BuildTree_r
- **Signature:** `node_t *BuildTree_r(node_t *node, bspbrush_t *brushes)`
- **Purpose:** Recursive single-threaded BSP builder. Selects a split side, partitions brushes, recurses into both children.
- **Inputs:** Current node (with pre-set volume brush), brush list.
- **Outputs/Return:** The completed subtree root.
- **Side effects:** Allocates child nodes; frees brush lists and volume brushes during AAS mode.
- **Calls:** `SelectSplitSide`, `LeafNode`, `SplitBrushList`, `FreeBrushList`, `AllocNode`, `SplitBrush`, `BuildTree_r` (recursive)

### BuildTreeThread / BuildTree
- **Signature:** `void BuildTreeThread(int threadid)` / `void BuildTree(tree_t *tree)`
- **Purpose:** Multi-threaded iterative equivalent of `BuildTree_r`. `BuildTree` initialises a work list (stack or queue), spawns threads, and waits. Each thread pulls nodes from the list via `NextNodeFromList`, processes them, and pushes child nodes back.
- **Side effects:** Thread locking/semaphore primitives; modifies global node list.
- **Calls:** `SelectSplitSide`, `LeafNode`, `SplitBrushList`, `SplitBrush`, `AllocNode`, `CheckBrushLists`, `FreeBrushList`, `FreeBrush`, `AddNodeToList`, `NextNodeFromList`, `AddThread`, `RemoveThread`, `WaitForAllThreadsFinished`

### BrushBSP
- **Signature:** `tree_t *BrushBSP(bspbrush_t *brushlist, vec3_t mins, vec3_t maxs)`
- **Purpose:** Top-level entry point. Validates input brushes, allocates the tree and head node, sets the head node's volume brush, then calls `BuildTree` (threaded iterative).
- **Inputs:** Full brush list, world bounds.
- **Outputs/Return:** Fully built `tree_t *`.
- **Side effects:** Logs statistics; resets counters; frees brush list internally via `BuildTree`.
- **Calls:** `Tree_Alloc`, `BrushVolume`, `AllocNode`, `BrushFromBounds`, `BuildTree`, `Log_Print`, `Log_Write`

## Control Flow Notes
`BrushBSP` is called once during map compilation (from `csg.c`/`bspc.c`) after CSG brush chopping. It is a compile-time (offline) operation, not part of the game runtime. The iterative `BuildTree` / `BuildTreeThread` path replaced `BuildTree_r` to support multi-core compilation; `BuildTree_r` is retained but no longer called in the active code path. The resulting `tree_t` is passed to portal generation and AAS creation stages.

## External Dependencies
- **Includes:** `qbsp.h` (all BSP types and constants), `l_mem.h` (memory allocation), `../botlib/aasfile.h` (AAS format constants), `aas_store.h` (`aasworld`, AAS types), `aas_cfg.h` (`cfg` for AAS expansion bbox check), `<assert.h>`
- **Defined elsewhere:** `mapplanes[]`, `numthreads`, `drawflag`, `create_aas`, `cancelconversion`, `microvolume` (globals from `bspc.c`/`map.c`); winding utilities (`BaseWindingForPlane`, `ClipWindingEpsilon`, `ChopWindingInPlace`, `WindingArea`, `WindingMemory`); thread primitives (`ThreadLock`, `ThreadSemaphoreWait`, `AddThread`, `RemoveThread`); GL debug draw (`GLS_BeginScene`, `GLS_Winding`, `GLS_EndScene`); `Tree_Alloc`; `Log_Print`/`Log_Write`
