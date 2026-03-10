# q3map/vis.c — Enhanced Analysis

## Architectural Role

This file implements the **PVS (Potentially Visible Set) compiler**, the core offline tool that computes inter-cluster visibility for a Q3 BSP. It consumes BSP portal/cluster data and outputs compact bitsets encoding which clusters can theoretically see each other—data consumed at runtime by the **renderer** (frustum/PVS culling), **server** (snapshot entity occlusion), and **qcommon CM module** (collision visibility queries). Without this preprocessing step, the engine would have no way to efficiently cull invisible geometry or entities.

## Key Cross-References

### Incoming (who calls this file's functions)
- **q3map/bspc.c** (`main`): Invokes `CalcVis()` after BSP construction as part of the full map compilation pipeline
- **q3map tool invocation**: End-user runs the tool with `-vis` flag; this drives the entire visibility phase

### Outgoing (what this file depends on)
- **q3map/threads.c** (`RunThreadsOnIndividual`): Parallelizes portal flow calculations across CPU threads
- **q3map/vis.h** (not shown): Declares portal/cluster structures and per-thread state
- **Common offline tool foundation** (`code/bspc/l_*.c`, `common/*.c`): memory, logging, filesystem, math utilities
- **Runtime consumers** (not direct calls, but data dependency):
  - `code/qcommon/cm_load.c`: Reads `visBytes` from compiled BSP via `CM_LoadMap`
  - `code/renderer/tr_world.c`: Uses `tr.world.clusterBytes` / `CM_ClusterPVS()` for view frustum culling
  - `code/server/sv_snapshot.c`: Uses PVS for entity snapshot occlusion culling

## Design Patterns & Rationale

**Multi-Pass Visibility Algorithm:**
- **BasePortalVis** (baseline, all portals see all): Coarse initial estimate, thread-safe initialization
- **CalcPortalVis** (portal-to-portal flow): Refines via iterative portal blocking tests; default mode
- **CalcPassageVis** / **CalcPassagePortalVis**: Advanced modes trading compute time for tighter bounds (fewer false positives)
- **FastVis**: Fallback for large maps; uses coarse mightsee heuristic to avoid quadratic complexity

**Bit-Vector Compression:**
- Portal and leaf visibility encoded as packed byte arrays (1 bit per entity, word-aligned)
- `leafbytes`, `leaflongs`, `portalbytes`, `portallongs` precomputed for quick indexing
- `ClusterMerge()` ORs portal visibility to produce leaf visibility, then compresses and writes to `visBytes`
- This reduces memory footprint from pointers to single bits, critical for large maps with 10,000+ portals

**Leaf Merging:**
- `TryMergeLeaves()` / `ClusterMerge()` coalesce geometrically adjacent clusters to reduce entity network overhead
- Portal/winding convexity tests prevent merging clusters separated by concave geometry

**Rationale:** Visibility precomputation is a classic game engine optimization. Q3 chose offline compilation to enable real-time renderer culling (PVS test is ~1 bit AND per cluster pair). Portal-based approach (not uniform grid) respects BSP geometry, reducing false positives.

## Data Flow Through This File

**Input:**
- BSP portal/cluster graph from prior `qbsp` phase (loaded into global `portals[]`, `leafs[]`)
- Winding geometry and plane equations for each portal

**Processing:**
1. `PlaneFromWinding()`: Extract plane equations from portal geometry (used for convexity tests)
2. `SetPortalSphere()`: Precompute bounding sphere for each portal (used in occlusion heuristics)
3. `SortPortals()`: Sort by `nummightsee` complexity (simple portals computed first, results reused)
4. **Phase selection** (fastvis / portal / passage modes):
   - `CalcPortalVis()` → `BasePortalVis()` + `PortalFlow()` (worker threads)
   - `CalcPassageVis()` → `CreatePassages()` + `PassageFlow()`
5. `ClusterMerge()`: Convert portal bitsets → leaf bitsets, ORing all connected portals
6. Write compressed bitsets to `visBytes` (offset `VIS_HEADER_SIZE + leafnum*leafbytes`)

**Output:**
- Global `visBytes[]` buffer, dumped into BSP file by caller; at runtime becomes `sv.clustervis` and `tr.world.clustervis`

## Learning Notes

**Idiomatic to this era / engine:**
- **Offline preprocessing as optimization**: Modern engines often use real-time software occlusion (HZB, compute shader rasterization). Q3's offline PVS is deterministic and tiny but inflexible (adding geometry requires recompilation).
- **Bit-packed data structures**: No SIMD or GPU compute available in 1999; bit arrays maximized cache efficiency on CPU.
- **Portal-based PVS vs. spatial grid**: Respects BSP geometry (fewer false positives than axis-aligned grids), but less parallelizable offline.
- **Multi-threaded worker model**: `RunThreadsOnIndividual()` predates OpenMP; manual work-stealing with per-thread state.

**Modern contrasts:**
- **ECS engines**: Would compute visibility on-demand via spatial hashing + frustum tests, no precomputation
- **Compute shaders**: Would parallelize occlusion across GPU; soft shadows, per-pixel visibility
- **Sparse voxel octrees / BVH**: Replace portal graphs; better hierarchical culling

**Design connections:**
- PVS is a **visibility aggregation** problem—classic in game engines (also used in light culling, shadow mapping, networking)
- Portal/cluster model is a hybrid between **BSP trees** (geometric partitioning) and **scene graphs** (hierarchical culling); more rigid than ECS but deterministic
- Leaf merging is a **space-filling optimization** (trading visibility precision for memory)

## Potential Issues

**Inferred from code:**
1. **Scalability**: Quadratic complexity in worst case (all-to-all portal tests). `FastVis` mitigates but produces loose bounds. Large maps (>10k portals) may timeout.
2. **Determinism assumptions**: `ClusterMerge()` and portal merging assume leaf/portal identity is stable across compilations; changes to BSP structure may invalidate cached visibility.
3. **Thread-safety in merge**: `TryMergeLeaves()` modifies portal `removed` flags; not guarded if called concurrently (appears single-threaded, but code review recommended).
4. **Passage visibility modes**: Not documented which mode produces tightest PVS; users must experiment (`-fastvis`, `-noPassageVis`, `-passageVisOnly`, `-mergevis` flags).
