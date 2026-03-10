# code/qcommon/cm_public.h — Enhanced Analysis

## Architectural Role

This header defines the **public collision subsystem API** that mediates between the high-level simulation (server game logic, physics, prediction) and the low-level BSP collision world. It is the sole interface through which all engine modules—server, game VM, cgame VM, and renderer—perform spatial queries, traces, and visibility tests. By exposing only opaque handles and immutable query results, it enforces a clean boundary around internal collision structures and allows the implementation to evolve independently.

## Key Cross-References

### Incoming (who depends on this file)

- **Server** (`code/server/sv_*.c`): Calls `CM_BoxTrace` for entity movement physics, damage radius tests (`G_RadiusDamage`); `CM_PointContents` for environmental hazards; `CM_LinkEntity` / sector queries for spatial partitioning
- **Game VM** (`code/game/g_*.c`): Entity movement, collision detection, melee/projectile traces; accesses via `trap_Trace` syscalls
- **cgame VM** (`code/cgame/cg_predict.c`): Client-side `Pmove` prediction using identical `CM_*` calls as server
- **Renderer** (`code/renderer/tr_world.c`): Calls `CM_ClusterPVS` each frame to cull invisible leaf clusters; builds draw-surface lists based on PVS visibility
- **Botlib** (`code/botlib/be_aas_*.c`): Indirectly via `botlib_import_t`; uses `CM_BoxTrace`, `CM_PointContents` for movement validation and reachability testing

### Outgoing (what this file depends on)

- **`qfiles.h`** (only `#include`): BSP file format structs (`dheader_t`, `dleaf_t`, `dnode_t`, `dbrush_t`, etc.) and map limits
- **Implementation modules**: Declarations here; bodies in `cm_load.c`, `cm_trace.c`, `cm_test.c`, `cm_patch.c`, `cm_tag.c`, `cm_marks.c` (all internal to qcommon)
- **Transitive type dependencies**: `clipHandle_t`, `trace_t`, `vec3_t`, `orientation_t`, `markFragment_t`, `qboolean` defined in `q_shared.h` or `cm_local.h` (private header)

## Design Patterns & Rationale

**Opaque Handle Abstraction**
- Functions return `clipHandle_t` (opaque integer or pointer), never expose internal model/brush structures
- Allows implementation to change collision data structures (e.g., from linear arrays to spatial trees) without breaking the API
- Seen in modern systems: Vulkan device/queue handles, graphics driver contexts

**Transform-Aware Duplication**
- Both `CM_BoxTrace` and `CM_TransformedBoxTrace` exist; same pattern for `CM_PointContents`
- Rationale: Submodels (rotating doors, moving platforms) have local-space geometry; transforms avoid redundant allocation of rotated copies
- Classic collision library design (PhysX, Bullet also offer both variants)

**Temporal Resource Reuse**
- `CM_TempBoxModel` returns a handle to a *single shared slot* that persists only until the next call
- Rationale: Avoids allocation overhead for temporary box models created every frame (e.g., predicted trajectory traces)
- Tradeoff: **Unsafe across thread boundaries** (documented)

**Modular Query Families**
- **Lifecycle**: `CM_LoadMap` (init), `CM_ClearMap` (shutdown)
- **Model Access**: `CM_InlineModel`, `CM_TempBoxModel`, `CM_ModelBounds`
- **Point/Box Queries**: `CM_PointContents`, `CM_BoxLeafnums`, `CM_PointLeafnum`
- **Visibility**: `CM_ClusterPVS`, `CM_NumClusters`
- **Connectivity**: `CM_AreasConnected`, `CM_AdjustAreaPortalState`, `CM_WriteAreaBits`
- **Specialized**: `CM_LerpTag`, `CM_MarkFragments`, `CM_DrawDebugSurface`
- Rationale: Separates concerns; allows incremental loading/updates; easier to test individual subsystems

**Mask-Based Content Filtering**
- `CM_PointContents` and trace results use ORed bitmasks: `CONTENTS_SOLID | CONTENTS_WATER | ...`
- `CM_BoxTrace` takes `brushmask` to selectively ignore certain contents
- Rationale: Enables flexible filtering without separate per-type trace functions

## Data Flow Through This File

**1. Map Load → World State**
```
CM_LoadMap("maps/q3dm1")
  → Loads BSP file from filesystem
  → Builds internal collision tree (AABBs, brush planes, leaf/node pointers)
  → Computes cluster PVS visibility data
  → Stores in module-global `cm_world` / `clipMap`
  → Returns checksum for integrity validation
```

**2. Trace Sweep → Collision Response**
```
CM_BoxTrace(start, end, mins, maxs, model, brushmask, capsule=0)
  → Traverses BSP tree (nodes → leaves → brushes)
  → Tests AABB/capsule against plane equations
  → Returns fraction [0..1] where 0 = no collision, 1 = full distance traveled
  → Also returns endpos, impact plane normal, surface flags, hit contents
```
Used by: physics engine (step down/up), damage traces, weapon/missile sweeps, prediction

**3. PVS Query → Visibility Culling**
```
CM_ClusterPVS(player_cluster)
  → Returns pointer to bitset (1 bit per world cluster)
  → Renderer iterates all visible leaf clusters, tests against this bitset
  → Culls surfaces in invisible clusters
```

**4. Area Connectivity → Sound/PVS Propagation**
```
CM_AdjustAreaPortalState(area1, area2, open=true)  // door opens
  → Marks portal as traversable
  → Affects audio propagation, snapshot generation (PVS flow)

CM_AreasConnected(area1, area2)
  → Checks if areas can "reach" each other through open portals
  → Used by server for audio distance modeling, snapshot filtering
```

**5. Model Interpolation → Animation Attachment**
```
CM_LerpTag(tag_out, model_handle, frame1, frame2, frac, "tag_weapon")
  → Interpolates MD3 tag (attachment point) between keyframes
  → Returns 3D position + orientation matrix
  → Used by cgame for weapon/flag positioning on player model
```

## Learning Notes

- **Offline Visibility Pre-computation**: Q3A's PVS (Potential Visibility Set) is computed by the BSP compiler offline, not at runtime. Modern real-time engines compute visibility dynamically (frustum, occlusion queries, hierarchical Z-buffer), but the PVS approach was extremely efficient for software rasterization / early OpenGL hardware.

- **Cluster-Based Spatial Partitioning**: Visibility and area connectivity operate at the *cluster* level (groups of leaves), not individual surfaces. This coarsening trades precision for speed—typical for Quake-era engines. Modern engines use per-pixel depth, BVH trees, or portal graphs.

- **Capsule Collision as Movement Option**: The `capsule` parameter allows choosing between AABB and capsule (cylinder with hemispherical ends) collision. Capsules better approximate humanoid bounding volumes and prevent "sticking" on corners. Modern physics engines default to capsules or convex hulls.

- **Transform-Aware Traces Avoid Duplication**: Rather than pre-rotating geometry or allocating temporary transformed models, the collision code applies inverse transforms at query time. Saves memory and allocator pressure but adds per-trace overhead.

- **Decal Clipping via MarkFragments**: `CM_MarkFragments` is a "decal projection" utility—clips a convex polygon (bullet hole, blast mark) against world surfaces to produce fragment polygons that exactly fit renderable surface patches. Modern engines often use deferred decals or dynamic atlasing instead.

- **Tag Interpolation for Attachments**: `CM_LerpTag` loads MD3 model tags (pre-authored attachment points like weapon slots, flag handholds) and interpolates them between animation frames. Modern skeletal animation systems integrate this more tightly; some engines use socket-based attachment (Unreal's sockets).

- **No VM Isolation at CM Level**: Game VM calls CM functions via syscalls, but CM itself runs in the engine process (not sandboxed). VM-level permission checks happen at the syscall dispatcher (`SV_GameSystemCalls`), not here.

## Potential Issues

- **Thread Safety of `CM_TempBoxModel`**: Creates a single reusable temporary model slot. If client and server run on separate threads (SMP mode enabled), simultaneous calls to `CM_TempBoxModel` + `CM_BoxTrace` could race. The handle would be invalidated mid-trace. Mitigation: Callers must serialize access or use thread-local temporary models.

- **No Map-Unload Synchronization**: `CM_ClearMap` deallocates the collision world with no refcounting or grace period. Pending traces on background threads (if any) would access freed memory. No documented safe shutdown sequence.

- **Capsule Mode Mismatch**: The `capsule` parameter must match between `CM_TempBoxModel(capsule=X)` and `CM_BoxTrace(capsule=X)`. If caller mixes them (create AABB, trace as capsule), results are undefined. No validation in this header.

- **Overflow in `CM_BoxLeafnums`**: The comment states "overflow if return listsize and if `*lastLeaf != list[listsize-1]`", but the header doesn't specify: (1) what happens to the output buffer if overflow occurs, (2) whether the function returns early or fills partially, (3) how much buffer space is *actually* needed. Callers must consult implementation.

- **No Eviction Policy for Area Portal State**: `CM_AdjustAreaPortalState` mutates global state. If gameplay code forgets to close a portal, sound and PVS will propagate through permanently. No automatic timeout or sanity checks.
