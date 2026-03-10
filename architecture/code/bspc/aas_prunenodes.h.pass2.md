# code/bspc/aas_prunenodes.h — Enhanced Analysis

## Architectural Role

This header declares the node-pruning stage of the **offline BSPC AAS compilation pipeline**. It sits within a multi-stage BSP→AAS transformation sequence that includes area creation, subdivision (gravitational/ladder), merging, edge-melting, face-merging, and finally pruning and optimization. Unlike the runtime bot navigation system in `code/botlib`, BSPC is a **standalone tool that transforms raw BSP geometry into a navigation graph**, and pruning is a critical quality-control gate that eliminates degenerate nodes before final serialization and runtime loading.

## Key Cross-References

### Incoming (who depends on this)
- **`code/bspc/bspc.c`** (main compiler entry): Invokes `AAS_PruneNodes()` as part of the offline compilation pipeline, likely after `AAS_Create()` and area merging/subdivision steps.
- **Compile-time pipeline stages** in BSPC: Called between geometry refinement (`aas_areamerging.c`, `aas_gsubdiv.c`) and the final optimization/output phases.

### Outgoing (what this file depends on)
- **Global AAS tree state** (implicitly): Operates on the in-memory `aasworld` singleton (same structure managed by `code/botlib/be_aas_main.c` at runtime, but modified offline).
- **`code/bspc/aas_prunenodes.c`**: Contains the actual implementation (`AAS_PruneNodes` + recursive `AAS_PruneNodes_r`).
- **Memory management**: Likely uses BSPC's zone/hunk allocator for node deallocation as redundant nodes are removed.

## Design Patterns & Rationale

**Single entry-point declaration pattern**: Typical of BSPC's `aas_*.h` headers—each module exports one or two public functions through a header, while implementation and internal helpers remain in the `.c` file. This mirrors botlib's separation of interface (`be_aas_*.h`) from implementation.

**Implicit global state**: Like all BSPC AAS functions, `AAS_PruneNodes()` receives no parameters and operates on the shared in-memory tree. This is efficient for offline tools (no malloc overhead per call) but would be problematic in a modern ECS or multi-threaded engine. BSPC's single-threaded, deterministic nature makes this acceptable.

**Placement in the pipeline**: Pruning happens *after* initial area creation and *before* optimization. This order is deliberate:
- Create areas from BSP geometry (potentially over-subdivided)
- Merge adjacent areas (reduce node count)
- Subdivide for movement types (ladders, gravity-affected zones)
- **Prune degenerate nodes** (eliminate unreachable or microscopically small areas)
- Optimize storage representation
- Write binary file

## Data Flow Through This File

```
[BSP tree] → [AAS_Create] → [Area merging] → [Subdivision]
                                                    ↓
                                              [AAS_PruneNodes] ← declaration here
                                                    ↓
                                         [Remove dead/tiny nodes]
                                                    ↓
                                              [AAS_Optimize]
                                                    ↓
                                         [AAS_WriteAASFile]
```

The function transforms the global AAS tree in-place, removing nodes that either:
- Have no reachable neighbors (orphaned)
- Are too small to pathfind through (below minimum presence bounds)
- Are created as artifacts of subdivision but never used

## Learning Notes

**Offline vs. runtime architecture**: Q3A cleanly separates the tool pipeline (BSPC) from the runtime library (botlib). This is a sound design choice for a game engine of that era—offline tools can be verbose and single-threaded, while runtime code is optimized. Modern engines often blur this line (e.g., Unity, Unreal) with scriptable editors, but Q3A's separation keeps compilation deterministic and reproducible.

**Why node pruning is essential**: Raw BSP→AAS conversion can produce a poorly connected graph with redundant nodes. Pruning ensures:
- Bots can reliably pathfind (no isolated nodes)
- File size is minimized (runtime AAS files are loaded into memory)
- Reachability queries are faster (fewer nodes to traverse)

**Contrast with modern engines**: Modern engines typically use spatial hashing or grid-based navigation (NavMesh), which naturally avoids degenerate states. Q3A's explicit node-pruning step reveals the maturity of its offline pipeline—it needed to account for BSP geometry edge cases that a hand-crafted navigation mesh wouldn't have.

## Potential Issues

None inferable from this declaration alone. The implementation in `aas_prunenodes.c` would reveal potential issues such as:
- Incomplete pruning (orphaned nodes remaining)
- Over-aggressive pruning (removing reachable nodes)
- Memory leaks if node deallocation is incomplete
- Infinite loops in recursive tree traversal if cycles exist (unlikely but possible with corrupt input)

These would only manifest at runtime during compilation of maps with pathologically complex geometry or corrupted BSP data.
