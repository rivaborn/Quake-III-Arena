# code/server/sv_world.c — Enhanced Analysis

## Architectural Role

This file implements the server's **spatial indexing and entity query subsystem**, forming a critical bridge between the game simulation (which must trace and query entities) and the collision system (which operates on static geometry). It maintains a uniform binary space-partition tree for fast entity culling and works hand-in-hand with the PVS cluster system to enable both gameplay mechanics (traces for shooting, movement collision) and visibility culling (snapshot generation only sends entities visible to the client).

## Key Cross-References

### Incoming (who depends on this file)

**From Game VM (via syscall dispatch in `sv_game.c`):**
- `trap_Trace` → `SV_Trace`: used extensively by `g_combat.c` (damage traces), `g_missile.c` (projectile movement), `g_active.c` (weapon firing), `g_move.c` (movement validation)
- `trap_EntityContactPoint` / point-contents variants → `SV_PointContents`: liquid/slime detection, environment damage tests
- `trap_EntitiesInBox` → `SV_AreaEntities`: spatial queries for item pickups, explosions, team scanning

**From Server frame-loop systems:**
- `sv_snapshot.c` reads `svEntity_t.clusternums[]` and `areanum` (populated by `SV_LinkEntity`) for PVS-based entity visibility culling
- `sv_main.c` calls `SV_ClearWorld` once during `SV_SpawnServer` after BSP load
- `sv_client.c` indirectly benefits from cluster caching via snapshot visibility

**From Bot integration:**
- Bots query the world through game VM syscalls, which delegate to these functions; `code/botlib` never calls this file directly

### Outgoing (what this file depends on)

**Collision Module (`code/qcommon/cm_*.c`):**
- `CM_InlineModel(modelindex)` → get BSP collision handle for inline models (brushes)
- `CM_TempBoxModel(mins, maxs, capsule)` → ephemeral collision models for non-bmodel entities
- `CM_ModelBounds(handle, mins, maxs)` → query world bounds at startup
- `CM_BoxTrace(trace, start, end, mins, maxs, model, contentmask)` → clip against static BSP
- `CM_TransformedBoxTrace(...)` → clip against rotated entity collision shapes (for dynamic entities)
- `CM_BoxLeafnums(mins, maxs, leafs, maxcount, lastLeaf)` → spatial query: which BSP leaves overlap box
- `CM_LeafArea(leafnum)`, `CM_LeafCluster(leafnum)` → convert leaf to area/cluster ID

**Server entity management:**
- `SV_SvEntityForGentity(gent)` → accessor: game entity ↔ server entity
- `SV_GEntityForSvEntity(sent)`, `SV_GentityNum(num)` → reverse mappings

**Math utilities:**
- `RadiusFromBounds(mins, maxs)` → compute bounding sphere radius for rotation expansion
- Vector macros: `VectorAdd`, `VectorSubtract`, `VectorCopy`

## Design Patterns & Rationale

**1. Uniform Axis-Aligned BSP for Spatial Partitioning**
- The sector tree is a **fixed-depth (4), uniform binary subdivision** splitting alternately on X and Y axes only.
- *Why uniform?* Guarantees bounded memory (max 16 leaves in a 64-node pool) and predictable traversal depth; avoids dynamic rebalancing overhead.
- *Why X/Y only?* Quake III maps are horizontally spread; vertical distribution (Z) is less critical for spatial queries.
- *Why axis-aligned?* Simplifies AABB overlap tests; no need to track plane normals like general BSP.
- *Alternative not used:* Octrees (more balanced) or dynamic BVHs (better for moving entities) would add complexity.

**2. Entity Linking as Lazy Geometric Indexing**
- Entities are stored in a linked list at the sector node that encloses or first spans their AABB.
- *Why lazy?* Entities are only relinked when they move (explicit `SV_LinkEntity` call by game), not every frame. Sector membership is **semi-static**.
- *Why linked lists at nodes?* Avoids fragmentation: an entity spanning two sectors is stored at their common ancestor, not duplicated.
- *Trade-off:* Traversal still requires AABB overlap tests even at leaf nodes; compensated by the small sector tree depth.

**3. Cluster/Area Caching at Link-Time**
- On link, `SV_LinkEntity` computes which PVS clusters and areas the entity occupies and caches them in `svEntity_t.clusternums[]` and `areanum`.
- *Why cache?* These values are read (not written) repeatedly by `sv_snapshot.c` during visibility culling. Computing them per-query would waste CPU.
- *Scope:* Up to `MAX_ENT_CLUSTERS` (usually ≤ 16) clusters per entity; overflow triggers a `lastCluster` fallback.
- *Key insight:* This decouples entity geometry updates from visibility updates — visibility is stale until the next link, but that's acceptable because moves are discrete game-frame events.

**4. Epsilon Expansion (1-unit AABB padding)**
- Every entity's absmin/absmax is expanded by 1 unit on all sides before any query or linking.
- *Why?* Movement is clipped an epsilon away from geometry edges; this padding catches edge cases where rounding might otherwise create slivers.
- *Game design implication:* The engine is "forgiving" — brush/entity boundaries have an implicit 1-unit tolerance zone.

**5. Solid Encoding in `entityState_t.solid`**
- `SV_LinkEntity` encodes the entity's bounding box geometry into a single 32-bit integer for client-side prediction.
- *Why send to client?* The cgame VM needs it to predict movement and detect collisions locally without waiting for server snapshots.
- *Encoding:* Upper 8 bits = max Z, middle 8 bits = min Z (negative), lower 8 bits = X/Y radius (assumed symmetric).
- *Simplification:* Only works for axis-aligned boxes; clients can't predict collisions against rotated bmodels locally (server-side only).

## Data Flow Through This File

**Initialization (map load):**
```
SV_SpawnServer (sv_main.c)
  → SV_ClearWorld (sv_world.c)
    → CM_InlineModel(0) + CM_ModelBounds → world AABB
    → SV_CreateworldSector (recursive) → populate sv_worldSectors[]
```

**Entity spawn/move:**
```
Game VM (G_SpawnEntity, G_MoveEntity) calls trap_LinkEntity
  → SV_GameSystemCalls (dispatch)
    → SV_LinkEntity (sv_world.c)
      → [if already linked] SV_UnlinkEntity (remove from old sector)
      → Compute AABB from origin + bounds; expand by 1 unit epsilon
      → CM_BoxLeafnums (query BSP to find overlapping leaves)
      → For each leaf: CM_LeafArea/CM_LeafCluster (populate cluster array)
      → Traverse sector tree: find first node that fully encloses entity
      → Insert into sector->entities linked list
      → Mark gEnt->r.linked = qtrue, increment gEnt->r.linkcount
      [PVS cluster info now cached; sv_snapshot.c will read later]
```

**Visibility culling (snapshot generation):**
```
SV_SendClientSnapshot (sv_snapshot.c, NOT this file)
  → reads svEntity_t.clusternums[] (computed by SV_LinkEntity)
  → CM_ClusterPVS (collision module)
  → Only includes entities visible from client's PVS
```

**Gameplay traces (movement, shooting):**
```
Game VM (G_Trace, projectile move, damage trace) calls trap_Trace
  → SV_GameSystemCalls → SV_Trace (sv_world.c)
    → CM_BoxTrace (clip against static BSP)
    → SV_ClipMoveToEntities (sv_world.c)
      → SV_AreaEntities (collect candidates in move envelope)
        → Traverse sector tree, collect entities whose AABB overlaps query region
      → For each candidate: SV_ClipToEntity
        → SV_ClipHandleForEntity → CM_TempBoxModel or CM_InlineModel
        → CM_TransformedBoxTrace (clip against entity's collision model)
        → Keep closest hit
    → Return combined trace result
```

## Learning Notes

**Visibility Architecture Insight:**
This file reveals that Q3's visibility system is **two-tier**: static BSP leaves are grouped into PVS clusters (precomputed offline by `q3map`), and **entity cluster membership is computed at runtime** (here). The game engine trusts that the BSP compiler has done the work; the server just caches membership. This is more efficient than per-entity PVS computation at query time.

**Why Not Modern Approaches?**
- **No Octree:** Would allow adaptive subdivision (finer in dense areas), but adds allocator overhead and rebalancing complexity unsuitable for 2005 code.
- **No Broad-phase AABB tree:** Physics engines (Havok, Bullet) use these for moving bodies, but Q3 is more static — sector updates are explicit, not continuous.
- **No Grid:** Fixed grids (like spatial hashing) are simpler but prone to boundary artifacts; the BSP approach is more elegant.

**Idiomatic Quake III Pattern:**
The explicit `SV_LinkEntity`/`SV_UnlinkEntity` API (vs. implicit spatial tracking) reflects the era's design: **entities are managed directly by game code**, not by a "physics engine." The server asks "link this entity here" rather than the engine asking "where is this entity?"

**Bridging Engine & VM:**
This file sits at a **critical syscall boundary**. Game logic in the VM issues trace requests; sv_world.c translates them into collision queries. The indirection allows the engine to swap collision backends without touching game code (though this didn't happen in Q3's lifetime).

**Interesting Omissions:**
- **No mutex/atomicity:** Assumes single-threaded server frame. Multi-threaded traces (on a dedicated server under high load) would race.
- **No profiling:** No per-function timing or query-count instrumentation visible, though production servers might benefit from knowing which entities cause the most traces.
- **No spatial coherence:** Unlike some game engines, there's no prediction that the next trace will be near the last one; each trace is independent.

## Potential Issues

**1. Sector Tree Overflow:**
If `sv_numworldSectors` exceeds `AREA_NODES` (64), subsequent allocations silently write out-of-bounds into `sv_worldSectors[]`. The recursion depth check (`AREA_DEPTH == 4`) prevents this on normal maps, but a pathological or hand-crafted BSP could trigger it.

**2. Linkcount Rollover:**
`gEnt->r.linkcount++` is never reset; on a very-long-running server, could theoretically overflow. Unlikely in practice (would require ~2 billion entity link events), but not documented.

**3. No Thread Safety:**
The sector tree and entity linked lists are unprotected. A multi-threaded server performing concurrent traces would race. Q3's architecture (single dedicated-server thread) doesn't hit this, but it's a latent hazard.

**4. Epsilon Expansion Loss of Precision:**
The 1-unit padding is absolute, not relative. For a tiny entity (e.g., 0.1 units), expansion to 1 unit is disproportionate; for a huge entity (1000 units), it's negligible. A relative epsilon might be more robust.

---

**Generated with Claude Code Architecture Analysis Pass 2**
