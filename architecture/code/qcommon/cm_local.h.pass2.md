# code/qcommon/cm_local.h — Enhanced Analysis

## Architectural Role

`cm_local.h` is the **private collision subsystem header** that underlies the qcommon core's BSP collision services. It acts as the integration point where the **Game VM** (`code/game`), **cgame VM** (`code/cgame`), **Server** (`code/server`), and **Renderer** (`code/renderer`) all converge to query the same precompiled world geometry. Each subsystem sees a different face of the CM API (`cm_public.h`), but internally they all read from the same `clipMap_t cm` global, accessed through shared qcommon infrastructure like `trap_Trace`, `CM_BoxBrushes`, and `trap_R_GetClusterPVS` (for renderer occlusion).

## Key Cross-References

### Incoming (who depends on this file)

- **Game VM** (`code/game/g_*.c`): Calls `trap_Trace` (→ `SV_Trace` → `CM_BoxTrace`), `trap_LinkEntity` (uses `cm` spatial structure), `CM_PointContents` for damage/lava/slime detection
- **cgame VM** (`code/cgame/cg_predict.c`): Calls `trap_Trace` to predict unacknowledged movement; reads `clipMap_t` leaf/surface data for sound propagation
- **Server** (`code/server/sv_world.c`): Manages the global `cm` load/unload; calls `CM_BoxBrushes`, `CM_PointContents` for physics queries; links/unlinks entities into the sector tree
- **Renderer** (`code/renderer/tr_world.c`): Calls `CM_ClusterPVS` (via qcommon's `Ri.CM_ClusterPVS` syscall wrapper) to cull surfaces outside the PVS; reads visibility `byte *` directly

### Outgoing (what this file depends on)

- **BSP File Format** (`cm_load.c` populates `clipMap_t` from `.bsp`): Reads `dshader_t`, plane arrays, node/leaf trees, brush definitions from `qfiles.h`
- **Math Utilities** (`q_math.c`, `q_shared.h`): `VectorCompare`, `CM_TraceThroughBrush` uses plane math, AABB operations
- **Memory Allocation** (`qcommon.h` `Hunk_Alloc`): All `clipMap_t` members are allocated from the hunk at load time
- **Patch Collision** (`cm_patch.c`): `patchCollide_s` (forward-declared, opaque at this header level) built by `CM_GeneratePatchCollide`, consumed by `CM_TraceThroughPatchCollide`

## Design Patterns & Rationale

1. **Precompiled World Geometry**: The entire `clipMap_t` is built once at load time and **never modified during gameplay**. This is idiomatic to offline/batch-compiled engines (maps are deterministic; no dynamic geometry). Contrast with modern ECS engines that stream geometry or support hot-swapping.

2. **Check-Count Stamps** (`cbrush_t.checkcount`, `cPatch_t.checkcount`): Rather than mark-and-sweep via bit-arrays or linked lists, the CM system uses a monotonically incrementing `cm.checkcount` global. Each trace/query increments this, and each brush/patch records the last check-count it was tested in. This avoids allocation during tight loops—a critical micro-optimization for 2000s-era hardware.

3. **Callback-Based Leaf Enumeration** (`leafList_t.storeLeafs`): The `CM_BoxLeafnums_r` recursion delegates leaf storage to a pluggable callback, allowing `CM_StoreLeafs` (for leaf queries) or `CM_StoreBrushes` (for brush collection) to be swapped without code duplication. This is a simple form of the **Strategy** pattern.

4. **Monolithic Global State**: All collision world data lives in one `clipMap_t cm` global. No handle/reference-counting—just a single active map per engine session. This simplifies the interface but couples all collision queries to the currently loaded map.

5. **Oriented Capsule as Degenerate Box**: The `sphere_t` struct parameterizes capsule traces within the general `traceWork_t` box-sweep framework, avoiding separate code paths. If `sphere.use` is true, traces are inflated along a center axis; otherwise, they degenerate to axis-aligned box sweeps.

## Data Flow Through This File

1. **Map Load** → `cm_load.c` parses BSP lumps → populates `clipMap_t cm` (planes, nodes, leafs, brushes, patches, visibility)
2. **Trace Query** → Game/cgame calls `trap_Trace` → fills stack `traceWork_t tw` with start/end/size → `CM_BoxTrace` iterates BSP tree, calls `CM_TraceThroughBrush`/`CM_TraceThroughPatchCollide` → updates `tw.trace` → returns to caller
3. **Point Query** → `CM_PointContents` descends BSP via `cNode_t` pointers, returns accumulated contents mask
4. **Area/Visibility** → `CM_ClusterPVS` returns visibility byte-string for PVS frustum culling; area portals gated by flood-fill stamping
5. **Map Unload** → `CM_ClearLevelPatches` frees patch collision structures; `cm` zeroed for next map load

## Learning Notes

**What's Idiomatic to Q3A Era:**
- **Offline BSP Compilation**: All geometry is precompiled; no realtime spatial indexing (no BVH, no octree rebuilding). Maps are static once loaded.
- **Per-Frame Check-Counts**: Avoids allocating/deallocating per-trace state; every object carries a "last tested in frame X" stamp.
- **Monolithic Struct (clipMap_t)**: Reflects pre-OOP design; contrast modern engines using ECS or scene graphs where collision is a component.
- **Explicit Visibility PVS**: Precomputed cluster visibility as byte-strings; modern engines use frustum culling + dynamic occlusion queries.

**Modern Engines Do This Differently:**
- **Per-Frame Allocators** instead of check-counts (Unity, UE5 use arena allocators resetting each frame).
- **Spatial Hashing / BVH Trees** for dynamic/deformable geometry; Q3A's static BSP tree is inflexible.
- **Component-Based** collision types (ECS capsules, boxes, meshes) vs. Q3A's monolithic `cbrush_t`/`cPatch_t`.
- **Streaming** large maps rather than loading the entire world into hunk memory.

## Potential Issues

1. **Global `cm` Coupling**: No map abstraction; all collision queries implicitly operate on the single active map. Multiplayer scenarios (split-screen, local bots) cannot have separate collision worlds.
2. **PVS Brittleness**: If a cluster boundary is misaligned with solid geometry, connectivity can be broken. Tools must validate; the runtime has no recovery.
3. **Patch Collision Overhead**: `patchCollide_s` is built per patch at load time; complex curved maps (many Bézier patches) bloat memory and slow initial load.
4. **No Dynamic Brushes**: All collision geometry is static; destructible geometry, expanding/shrinking brushes, or moving platforms must be hand-coded per entity type (movers, doors) rather than expressed in the CM system.

---

**Key Takeaway**: `cm_local.h` exemplifies how **precompiled, static spatial data** (BSP tree, PVS, brush geometry) can service high-performance collision queries across multiple engine subsystems with minimal runtime overhead—a hallmark of Q3A's efficient design, though inflexible by modern standards.
