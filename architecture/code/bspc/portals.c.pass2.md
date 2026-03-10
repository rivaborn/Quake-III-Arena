# code/bspc/portals.c — Enhanced Analysis

## Architectural Role
This file implements the **portalizing phase** of the offline BSPC (BSP→AAS compiler) pipeline. It bridges the tree-building phase (`BrushBSP`) and navigation-mesh generation, converting a raw BSP tree into a portal-annotated structure. Portals serve as the intermediate representation: convex polygon boundaries between leaf nodes that enable both visibility (PVS) and reachability (AAS) computation. The dual flood-fill pattern—entity-driven spatial marking followed by content-based area classification—is the critical step that transforms a geometric BSP into a navigationally meaningful scene partition.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** – Main BSPC entry point drives the overall compilation pipeline: after `BrushBSP` builds the tree, it calls `MakeTreePortals` → `FloodEntities` → `FillOutside` → `FloodAreas` → `MarkVisibleSides` in sequence
- **Compile flow**: Portals are created before AAS cluster/reachability computation in `code/bspc/be_aas_bspc.c:AAS_CalcReachAndClusters`

### Outgoing (what this file depends on)
- **Winding operations** (from `l_poly.h`): `BaseWindingForPlane`, `ChopWindingInPlace`, `ClipWindingEpsilon`, `FreeWinding`, `WindingIsTiny` — all perform convex polygon clipping
- **Global BSP state** (`qbsp.h` via `code/bspc/qbsp.h`):
  - `mapplanes[]` – Plane array indexed by node plane numbers; clipping uses plane normals and distances
  - `entities[]`, `num_entities` – Entity list (e.g., `info_player_start`, movers) for entity flood-fill origin detection
  - `numthreads` – Single-thread check for memory/counter tracking
- **Memory layer** (`l_mem.h`): `GetMemory`, `FreeMemory`, `MemorySize` for portal allocation with bookkeeping
- **Utility macros** from `q_shared.h`: `VectorSubtract`, `VectorCopy`, `DotProduct`, `ClearBounds`, `AddPointToBounds`

## Design Patterns & Rationale

### Flood-Fill Duality
- **Entity flood** (BFS via queue): Starts from entity origins (e.g., spawn points), marks `node->occupied` on all reachable non-solid leaves. This identifies the "playable" part of the level.
- **Content flood** (DFS): `FillOutside` recursively marks unreachable leaves as `CONTENTS_SOLID`, creating a boundary. `FloodAreas` then assigns integer IDs to the remaining reachable regions, stopping at `CONTENTS_AREAPORTAL` nodes (manual portals set by the level designer).
- **Rationale**: Splitting entity reachability from content-based classification allows both gameplay logic (where players can go) and designer intent (area portal placement) to influence the final result.

### Portal as Intermediate Representation
Portals are lightweight convex polygons (windings) rather than expensive full-mesh structures. Each portal is clipped iteratively against ancestor planes (`BaseWindingForNode`), then against sibling portals (`MakeNodePortal`), producing tight-fit boundaries. This is memory-efficient and deterministic.

### Single-Thread Telemetry
Memory/counter tracking (`c_active_portals`, `c_peak_portals`, `c_portalmemory`) is gated on `numthreads == 1`. This pattern reflects the era: multithread compile-time parallelism existed but was not always enabled. The tracking itself is too expensive to always run.

## Data Flow Through This File

```
Input:  BSP tree from BrushBSP (node->children, plane linkage, brush occupancy)
        Entity list (spawn points, movers, jump pads)
        Map plane array (normalized half-space definitions)

Phase 1: MakeTreePortals_r (DFS post-order traversal)
  → At each internal node: create a portal on its splitting plane,
    clip it against parent/sibling portals, then split for children
  Output: tree fully portalyzed, each node has linked-list portal set

Phase 2: FloodEntities (BFS from entity origins)
  → Start from playable entities (info_player_start, etc.)
  → BFS through non-solid portals, marking node->occupied
  → Nudge spawn points ±16 if initial placement fails (level-editor robustness)
  Output: node->occupied flags set on all reachable leaves

Phase 3: FillOutside_r (DFS)
  → Recursively mark unoccupied non-solid leaves as CONTENTS_SOLID
  → Isolates the "outside" void
  Output: void space is now solid; only playable space remains

Phase 4: FloodAreas_r (DFS with portal traversal)
  → Assign incrementing area IDs to reachable leaves
  → Stop at CONTENTS_AREAPORTAL nodes (designer-placed portals)
  → Record which two areas each areaportal connects
  Output: node->areaID populated; areaportal linkage established

Phase 5: MarkVisibleSides_r (DFS with brush side visibility)
  → For each non-empty leaf, find best-matching brush side (plane alignment)
  → Set SFL_VISIBLE on that side
  Output: Brush sides marked for BSP output (skips fully-occluded geometry)
```

## Learning Notes

### Idiomatic to This Era
- **Portal as first-class entity**: Modern engines often merge portals into the visibility system (PVS matrix, hierarchical Z-buffer). Q3 uses explicit portal polygons for both PVS and entity reachability, making them visible in the BSP structure.
- **Entity flood as "outside" detector**: Rather than a global OUTSIDE_NODE flag, the engine floods from entity origins. Any unreached leaf is inferred to be "outside" and is filled with solid. This is practical and robust.
- **Tiny portal culling**: Degenerate portals (from numerical clipping) are detected and discarded. This prevents downstream AAS fragmentation from near-zero-area geometry.
- **BFS for entity flood, DFS for area flood**: The choice reflects the different concerns—entity reachability is a graph connectivity problem (BFS), while area ID assignment is a tree traversal (DFS).

### Connection to Modern Game Engine Concepts
- **Portal-based visibility**: Precursor to modern **portal culling** (e.g., Unreal Engine's room/portal graphs). Here, portals are lightweight and tightly coupled to the BSP tree.
- **Reachability graph**: The areaportal records connect to **navigation mesh** concepts in modern engines (NavMesh links, warp portals).
- **Flood-fill classification**: Similar to **spatial region labeling** in modern level design tools, but deterministically computed from BSP structure rather than user-drawn boundaries.

## Potential Issues

1. **Fixed-size portal stack** (line ~265): `RemovePortalFromNode` declares `portal_t *portals[4096]` for circular-link detection. If a leaf has >4096 portals, the check silently skips (commented `//if (++n >= 4096)...`). In practice, Q3 maps rarely exceed this, but it's a latent overflow risk.

2. **Spawn point nudging** (implicit in `FloodEntities`): If `info_player_start` initial placement fails, the code nudges ±16 units. This works but is fragile; level designers can still author broken spawn points that won't be detected until the engine loads the map.

3. **Entity flood requires exact leaf checks** (line ~200): `Portal_EntityFlood` errors if nodes are not `PLANENUM_LEAF`. This is correct (only leaves have contents), but makes the code fragile to any upstream tree-building bugs that create non-leaf portals.

4. **No explicit validation of portal windings** (line ~320): The `#ifdef DEBUG` block that would check `WindingError` is commented out. Clipping can introduce numerical errors; silent silent geometry is worse than early error detection.

5. **Areaportal linkage assumes exactly two areas**: `FloodAreas` assumes each areaportal connects precisely two regions. If a designer places a portal that touches >2 areas (e.g., in a room corner), behavior is undefined.
