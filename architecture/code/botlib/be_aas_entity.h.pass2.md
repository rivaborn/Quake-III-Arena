# code/botlib/be_aas_entity.h — Enhanced Analysis

## Architectural Role

This header declares the **entity facade** layer of the AAS subsystem within botlib. It bridges the game/server world (where entities have dynamic positions, velocities, and collision state) to the bot pathfinding layer (which must map those entities into AAS topology for goal selection and reachability queries). The AASINTERN-gated functions implement a **mark-and-sweep per-frame entity lifecycle**, while the public API enables on-demand spatial queries during bot decision-making.

## Key Cross-References

### Incoming (who depends on this file)
- **code/game/g_bot.c** / **code/server/sv_bot.c**: Bot lifecycle management; likely calls `AAS_StartFrame` cycle (which internally triggers `InvalidateEntities` → game pushes `UpdateEntity` calls)
- **code/game/ai_dmq3.c** / **code/game/ai_dmnet.c**: Bot AI decision-making calls `AAS_BestReachableEntityArea()` during goal selection (weapon pickup, enemy, flag, defend point). Also queries entity properties via `AAS_EntityOrigin()`, `AAS_EntityType()` for threat assessment and movement planning.
- **code/botlib/be_aas_route.c**: Reachability and routing subsystem uses `AAS_BestReachableEntityArea()` to anchor pathfinding queries to entity-space targets.
- **Implicit botlib API boundary**: The `botlib_import_t` vtable (provided by server at initialization) routes entity updates from game → botlib entity system.

### Outgoing (what this file depends on)
- **code/botlib/be_aas_entity.c**: Contains all function implementations; this header is its public declaration.
- **code/botlib/be_aas_sample.c**: Implementation likely calls `AAS_PointAreaNum()`, `AAS_TraceClientBBox()` (spatial queries to convert entity positions to AAS areas).
- **code/botlib/be_aas_bspq3.c**: `AAS_EntityBSPData()` implementation retrieves BSP collision data for entities, required for trace/clip operations during movement prediction.
- **code/botlib/be_aas_bsp.h**: Defines `bsp_entdata_t` type used by `AAS_EntityBSPData()`.
- **code/botlib/be_aas_def.h**: Defines `aas_entityinfo_t` type returned by `AAS_EntityInfo()`.
- Indirectly: **qcommon** globals and services accessed via `botimport` vtable (memory alloc, PVS checks, file I/O).

## Design Patterns & Rationale

**Mark-and-Sweep Frame Lifecycle**  
The triplet `InvalidateEntities()` → (game pushes `UpdateEntity()` calls) → `UnlinkInvalidEntities()` is a classic **safe state transition pattern**. At frame start, all entities are marked invalid. Game updates only touched entities; unlinked ones are swept away. This avoids stale pointers and dead-entity ghosts in AAS area link lists.

**Facade / Service Locator**  
Public getters (`EntityOrigin`, `EntitySize`, `EntityType`, etc.) hide the global `aasworld` entity table. Callers never directly manipulate entity state; all queries go through this facade, enabling safe refactoring of the internal representation.

**Separation of Concerns: AASINTERN vs Public**  
The `#ifdef AASINTERN` guard restricts lifecycle functions (invalidate, update, reset) to internal use. Public callers only see read-only accessors. This is a C-era pattern mimicking private/public without language support—enforced by convention and build configuration.

**Why This Structure?**  
- **Encapsulation**: Entity state is strictly managed by AAS; game cannot directly corrupt it.
- **Per-Frame Coherence**: Frame boundaries are explicit, preventing mid-frame inconsistencies.
- **Determinism**: Bot AI sees a snapshot of entity state; no race conditions or asynchronous updates.

## Data Flow Through This File

```
Frame N Start:
  AAS_InvalidateEntities()           [mark all as potentially stale]
  
Game Logic Runs (Server Frame):
  For each entity needing AAS update:
    AAS_UpdateEntity(ent_id, state)  [recompute spatial links]
    
Frame Boundary:
  AAS_UnlinkInvalidEntities()        [sweep: remove untouched entities from area lists]
  
Bot AI (concurrent with next frame setup):
  AAS_BestReachableEntityArea(ent_id)  → AAS area index for pathfinding
  AAS_EntityOrigin(ent_id, out_vec)    → world-space position
  AAS_EntitySize(ent_id, mins, maxs)   → bounding box for collision checks
  AAS_EntityInfo(ent_id, info_out)     → full snapshot for detailed queries
```

Entity positions flow from **game physics → AAS entity table → bot goal selection → pathfinding routing**. The entity system is **read-mostly** from bot AI's perspective (except during frame setup).

## Learning Notes

**Idiomatic 2000s AAA Engine Design**  
This file exemplifies pre-ECS, pre-data-driven architecture:
- Explicit entity IDs (integers) instead of handles or references
- Monolithic entity array accessed via syscalls
- No batch queries or hierarchical spatial structures in the public API
- Imperative lifecycle (InvalidateEntities, UpdateEntity, UnlinkInvalidEntities) vs. event-driven modern approaches

**Determinism vs. Convenience Trade-off**  
The mark-and-sweep pattern ensures bot AI never sees intermediate/partially-updated entity state—critical for **deterministic multiplayer replays and network consistency**. Modern engines often accept eventual consistency for single-player convenience; Quake III prioritizes replay/netcode fidelity.

**Spatial Bridging Challenge**  
Games need to map world-space entity positions (continuous, physics-based) to discrete navigation graphs (AAS areas). `AAS_BestReachableEntityArea()` is the lynchpin—it likely finds the nearest walkable area to an entity's position, handling edge cases like entities on slopes, platforms, or ladders. This is a **non-trivial geometric problem** that modern engines often solve with persistent spatial indexes; Q3 recomputes on-demand.

## Potential Issues

- **Entity Index Bounds**: Functions accept `int entnum` with no declared range. Callers must know max entity count (likely defined in `game/g_local.h`). Buffer overruns possible if indices stray.
- **Stale Pointer Risk in AASINTERN Functions**: If `AAS_ResetEntityLinks()` is called mid-frame while bot AI holds cached `aas_entityinfo_t` pointers, use-after-free is possible. Mitigated by strict frame-boundary calling convention, but no language-level enforcement.
- **O(N) Entity Iteration**: `AAS_NextEntity()` likely scans linearly. With 1024+ entities, per-frame iteration overhead could become noticeable (though Quake III typically caps at ~256 entities in gameplay).
- **No Validation**: Public accessors (`AAS_EntityOrigin`, `AAS_EntitySize`) assume valid entity indices. Passing invalid IDs silently returns garbage or crashes; no error codes.
- **Implicit Coupling to Game VM**: The entity update protocol (how game sends updates to botlib) is not visible in this header—it's hidden behind `botlib_import_t` and syscall dispatch. Changes to `bot_entitystate_t` can silently break AAS logic.
