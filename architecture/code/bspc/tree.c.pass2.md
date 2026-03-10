# code/bspc/tree.c — Enhanced Analysis

## Architectural Role
This file manages the **offline BSP tree lifecycle** during map compilation in BSPC—a completely separate tool from the runtime engine. It bridges the spatial-subdivision construction phase (CSG, face generation) and the optimization/output phase by providing allocation, recursive traversal, pruning, and deallocation. Unlike the runtime `code/botlib` (which holds *fixed* compiled navigation meshes), this tree is **transient**: created during compilation, optimized via `Tree_PruneNodes`, then freed when the compiler exits.

## Key Cross-References

### Incoming (who depends on this file)
- **BSPC compiler phases** (csg.c, faces.c, map.c, brushbsp.c) → construct BSP nodes and invoke tree functions
- **`be_aas_bspc.c`** (botlib reuse in BSPC context) → calls `AAS_Create` and related area generation, which rely on a well-pruned tree
- **Pruning pipeline** → `Tree_PruneNodes` is invoked post-construction to optimize the tree before AAS extraction
- **Leak detection / flood-fill passes** → use `NodeForPoint` to locate leaf nodes for flood fill

### Outgoing (what this file depends on)
- **Memory management** (`l_mem.c`): `GetMemory`, `FreeMemory`, `MemorySize` (debug only)
- **Portal management** (`portals.c`): `RemovePortalFromNode`, `FreePortal` — bidirectional portal cleanup
- **Brush management** (`brushbsp.c`): `FreeBrush` — frees individual brush references
- **Math utilities** (`l_math.h`): `ClearBounds` for tree bounds initialization
- **Logging** (`l_log.c`): `Log_Print`, `PrintMemorySize` — optional debug reporting
- **Global state** (`qbsp.h`): `mapplanes[]` array (plane database), `PLANENUM_LEAF` constant, `create_aas` flag, `numthreads`

## Design Patterns & Rationale

**Two-Phase Deallocation**: `Tree_Free` deliberately calls `Tree_FreePortals_r` *before* `Tree_Free_r` because portals maintain bidirectional links between nodes (`p->nodes[0]` and `p->nodes[1]`). Freeing nodes first would orphan portal references.

**Recursive Tree Collapse**: `Tree_PruneNodes_r` collapses interior nodes where *both* children are `CONTENTS_SOLID` into a single leaf. This optimization reduces tree depth and memory footprint post-construction—a classic spatial-index optimization from the Quake era.

**Plane-Index Traversal**: `NodeForPoint` uses a plane sign test (`DotProduct` vs `plane->dist`) to classify points without storing explicit child pointers per plane—a compact representation inherited from BSP compiler lineage.

**AAS Ladder Exemption**: Pruning skips collapse if either child contains `CONTENTS_LADDER` (when `create_aas` is set) because the AAS reachability system needs explicit ladder geometry for pathfinding, not a collapsed solid leaf.

## Data Flow Through This File

1. **Construction** (BSPC pipeline) → `Tree_Alloc` creates empty tree; construction phases populate nodes/portals/brushes
2. **Traversal** (leak detection) → `NodeForPoint` descends tree to find leaf containing a 3D point (used by flood-fill)
3. **Optimization** → `Tree_PruneNodes` collapses redundant solid-solid interior nodes, merges their brush lists
4. **Deallocation** → `Tree_Free` recursively frees portals, brushes, volumes, and nodes; accumulates freed bytes for logging
5. **Output** → Optimized tree feeds into AAS area generation (`AAS_Create` in `aas_create.c`)

## Learning Notes

- **Idiomatic to offline 1990s BSP compilers**: The recursive fixed-tree approach predates modern spatial acceleration structures (octrees, BVH). This reflects the era when compile-time overhead was acceptable for well-structured BSP.
- **Portal duality** is a key concept: each portal connects two nodes bidirectionally, so cleanup must handle both directions.
- **The `PLANENUM_LEAF` sentinel** is foundational to tree traversal—every algorithm depends on detecting leaf nodes via this constant, not a pointer check.
- **Debug instrumentation via `#ifdef ME`** suggests legacy developers (possibly one engineer's custom memory tracking). Modern engines would use standard profilers.
- **Single-threaded node counting** (`if (numthreads == 1) c_nodes--`) hints that multithreaded BSPC runs skip this counter—a workaround for thread-unsafe global state.

## Potential Issues

- **Memory leak risk** if `c_nodes` counter diverges from actual allocation—leaked if a `Tree_Free` path is bypassed.
- **Portal cleanup order is critical**: calling `FreePortal` without first unlinking via `RemovePortalFromNode` could double-free or access freed memory.
- **Implicit `#ifdef ME` debug path**: shipping code with `#ifdef ME` dead code is a code-smell; this should be cleaned up or made opt-in via cvar.
- **AAS-tree coupling**: The `create_aas` flag couples this generic tree module to AAS specifics (ladder content); future re-use would require parameterization.
