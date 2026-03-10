# code/qcommon/cm_test.c — Enhanced Analysis

## Architectural Role

This file is the **spatial query gateway** for the collision map subsystem. It bridges raw BSP tree traversal (finding leafs) with higher-level semantic queries (what's at a point, what's in a box, are two areas connected). The area connectivity system here directly feeds the server's snapshot culling pipeline: `CM_WriteAreaBits` output determines which entities each client receives, making this code critical to network bandwidth optimization. It also exposes PVS cluster data to the renderer for frustum-culling visibility.

## Key Cross-References

### Incoming (who depends on this file)
- **Server snapshot generation** (`code/server/sv_snapshot.c`): Calls `CM_AreasConnected` and `CM_WriteAreaBits` to compute per-client visibility and cull non-visible entities before transmission
- **Renderer** (`code/renderer/tr_world.c`, `tr_main.c`): Calls `CM_ClusterPVS` with cluster indices from BSP traversal to determine which world surfaces are potentially visible
- **Game VM** (`code/game/g_*.c`): Calls `CM_PointContents` and `CM_TransformedPointContents` to query what medium entities occupy (water, lava, etc.) for damage and status effects
- **Collision dispatcher** (`code/qcommon/cm_trace.c`): Shares the same `cm` global and BSP traversal patterns; this file is the point-query sibling to trace's sweep-query logic
- **Area portal door logic** (`code/game/g_*.c`): Calls `CM_AdjustAreaPortalState` when doors open/close to trigger full connectivity recompute

### Outgoing (what this file depends on)
- **BSP data structures** (loaded by `code/qcommon/cm_load.c`): `cm.nodes`, `cm.leafs`, `cm.brushes`, `cm.areas`, `cm.areaPortals`, `cm.visibility` globals
- **Math utilities** (`code/qcommon/q_shared.c`, `q_math.c`): `DotProduct`, `VectorCopy`, `VectorSubtract`, `AngleVectors`, `BoxOnPlaneSide`
- **Core engine services** (`code/qcommon/qcommon.c`): `Com_Error`, `Com_Memset`
- **CVar system**: Reads `cm_noAreas` for debug bypass (registered elsewhere, e.g., `code/qcommon/cm_main.c`)
- **VM hosting**: `CM_ClipHandleToModel` → resolves submodel handles used by `CM_PointContents`

## Design Patterns & Rationale

**Callback-driven enumeration** (`leafList_t.storeLeafs` function pointer):  
Rather than duplicate the BSP recursion logic, `CM_BoxLeafnums_r` accepts a callback (`CM_StoreLeafs` or `CM_StoreBrushes`) to accumulate results. This is a lightweight alternative to templates or virtual dispatch, fitting Q3's constraints.

**Reference-counted portals with lazy recompute**:  
`CM_AdjustAreaPortalState` increments/decrements counters instead of immediately recomputing all connectivity. `CM_FloodAreaConnections` recomputes *all* areas in one pass when a change occurs. This batches expensive flood-fill work and supports multiple simultaneous door openers.

**Global checkcount for deduplication** (`cm.checkcount`):  
Rather than maintain a per-brush visited set across leaf enumeration, the code increments a global counter and stamps each brush. When the counter wraps (theoretically), all stamps are reset. This is cache-friendly and requires no dynamic allocation.

**Axial-plane fast path** (`plane->type < 3`):  
`CM_PointLeafnum_r` avoids expensive dot products for axis-aligned planes. This reflects the era's CPU constraints and the fact that most map brushes are axis-aligned.

## Data Flow Through This File

**Point-in-contents query** (high-frequency):  
Game code → `CM_TransformedPointContents` (transforms point) → `CM_PointContents` → `CM_PointLeafnum_r` (BSP walk) → iterate leaf's brushes → test point against brush planes → ORed content flags (integer) → back to game (used to determine damage type, medium).

**Area visibility export** (per-client, per-snapshot):  
Server snapshot builder → `CM_WriteAreaBits` (takes client's area) → checks `cm_noAreas` cvar or computes flood-connected areas → writes bitmask into packet → client-side cgame uses to filter entity snapshots.

**Portal state update** (event-driven):  
Game code detects door open → `CM_AdjustAreaPortalState` → increments portal refcount → triggers `CM_FloodAreaConnections` → full flood-fill to recompute all `area.floodnum` values → next snapshot uses updated connectivity.

## Learning Notes

**Why this architecture works for Q3**:  
- BSP-centric spatial decomposition made area connectivity (via portals) a natural fit for coarse-grained visibility culling
- Reference-counted portals allow doors to block connectivity without special state machines
- Global checkcount is cache-friendlier than per-entity visited flags in tight loops
- Callback pattern avoids code duplication despite different accumulation strategies
- PVS exposure to renderer enables frustum culling inside visible clusters only

**Modern engines do differently**:  
- Scene graphs / octrees / BVH hierarchies replace static BSP for dynamic worlds
- Spatial culling is often data-driven (bitmask queries) rather than callback-based
- Visibility computation (PVS) is offline-baked; runtime doesn't compute reachability
- Networking uses interest-based culling (distance, type) rather than area-based

**Idiomatic patterns of the Q3 era**:  
- Global singleton (`cm`) loaded once at map startup, queried by all subsystems
- Mixed offline/online work (PVS baked, area connectivity recomputed at runtime)
- Stateful iteration using function pointers instead of lambda/closure syntax
- Tactical optimizations (axial fast path, global counter) visible throughout

## Potential Issues

1. **Floating-point precision in `CM_PointContents`** (line 246): The `> dist` vs `>= dist` FIXME suggests points exactly on brush boundaries may yield inconsistent results across slight floating-point jitter. This could cause entities to clip through walls under edge-case conditions.

2. **Area flood-fill cycle detection** (`CM_FloodArea_r`): The `Com_Error` on `reflooded` indicates the algorithm assumes no cycles in the area-portal graph. A malformed map could trigger an assertion; no graceful fallback.

3. **Performance of `CM_WriteAreaBits` on large area counts**: The bit-loop (lines 402–407) is O(numAreas); on maps with hundreds of areas, per-client snapshot generation becomes a bottleneck. Modern engines might use precomputed visibility matrices or GPU-accelerated culling.

4. **No bounds checking in `CM_StoreLeafs` / `CM_StoreBrushes`**: Overflow sets `ll->overflowed` but continues processing. Caller must check the flag; silent list truncation is possible if the flag is ignored.
