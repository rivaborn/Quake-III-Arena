# code/cgame/cg_ents.c — Enhanced Analysis

## Architectural Role

This file is the **client-side entity presentation layer**, consuming `clSnapshot_t` packets and transforming them into renderer and audio subsystem commands. It sits between the network layer (snapshot acquisition in `cg_snapshot.c`) and the output subsystems (renderer, sound), implementing per-frame interpolation, type-specific rendering dispatch, and continuous effects (lighting, audio). As the sole per-frame entry point for all packet entities (`CG_AddPacketEntities`), it drives the majority of visible gameplay on the client.

## Key Cross-References

### Incoming (who depends on this)
- **`CG_DrawActiveFrame` (cg_view.c)** → calls `CG_AddPacketEntities()` once per rendered frame, the main entry point
- **Renderer subsystem** → receives `refEntity_t` via `trap_R_AddRefEntityToScene()` for all visible entities and their effects
- **Sound subsystem** → receives looping sound registrations and spatial position updates via `trap_S_*` traps
- **Tag attachment consumers** → `CG_Player` (cg_players.c) and weapon rendering code use `CG_PositionEntityOnTag/CG_PositionRotatedEntityOnTag` to attach child models to bone positions

### Outgoing (what this file depends on)
- **`cg_players.c`** → `CG_Player()` handler for skeletal entity rendering (called from `CG_AddCEntity`)
- **`cg_weapons.c`** → `weapon->missileTrailFunc` (e.g., `CG_GrappleTrail`) invoked by `CG_Missile`; weapon metadata (`cg_weapons`)
- **`bg_pmove.c`/`bg_misc.c`** → `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta`, `BG_PlayerStateToEntityState` for physics and state transformation
- **Renderer traps** → `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_LerpTag` for tag interpolation
- **Sound traps** → `trap_S_UpdateEntityPosition`, `trap_S_AddLoopingSound`, `trap_S_AddRealLoopingSound`, `trap_S_StartSound`
- **Global state** → `cg` (frame state, time, interpolation), `cgs` (cached models, sounds, item/weapon info), `cg_entities` (per-entity centity_t array)

## Design Patterns & Rationale

**1. Type Dispatch via Switch (CG_AddCEntity)**
   - Avoids virtual function overhead in a QVM environment; simple linear switch is cache-friendly and deterministic.
   - Early-exit for event types (`ET_EVENTS`) keeps event handling separate (belongs in `cg_event.c`).

**2. Per-Frame Interpolation State (cg.frameInterpolation)**
   - Computed once in `CG_AddPacketEntities`, used by all calls to `CG_CalcEntityLerpPositions`.
   - Decouples the interpolation fraction (network/server timing) from per-entity lerp logic.

**3. Shared Auto-Rotation Axes (cg.autoAngles/autoAxis)**
   - Computed once per frame for all world items to avoid redundant sin/cos per item.
   - Two rotation speeds (fast/normal) for aesthetic variety without branching per item.

**4. Tag-Based Hierarchical Attachment**
   - `CG_PositionEntityOnTag`: replaces child axis (used for weapons floating in world).
   - `CG_PositionRotatedEntityOnTag`: preserves child's pre-existing rotation (used for barrel/muzzle flash on rotating projectiles).
   - Delegates bone interpolation to renderer via `trap_R_LerpTag`, avoiding duplication.

**5. Mover Adjustment Isolation**
   - `CG_AdjustPositionForMover` applies platform/elevator correction; called only for non-predicted entities.
   - Predicted player skips this to avoid double-correction (server and client both move the player).

**6. Continuous Effects Decoupling (CG_EntityEffects)**
   - Looping sound, dynamic lights, and sound position updates run separately from entity rendering.
   - Allows entities to be culled from rendering but still play audio (e.g., offscreen weapon fire sound).

## Data Flow Through This File

```
Server Snapshot (clSnapshot_t)
    ↓
CG_AddPacketEntities()
  • Compute frameInterpolation
  • Seed auto-rotation axes (global state)
  • Add predicted player entity
  ↓
  For each snapshot entity:
    ↓
    CG_AddCEntity()
      ↓
      CG_CalcEntityLerpPositions()
        • Interpolate origin/angles via trajectory evaluation
        • Apply mover platform offset (if not predicted)
      ↓
      CG_EntityEffects()
        • Update sound 3D position
        • Add looping sounds, dynamic lights
      ↓
      Type Handler (CG_Item, CG_Missile, CG_Player, etc.)
        • Transform lerped state → refEntity_t/audio commands
        • Dispatch to renderer/sound subsystem
      ↓
Renderer Command Queue, Sound Queue
    ↓
Rendered Frame, Mixed Audio
```

**Key state mutations:**
- `cg.frameInterpolation`, `cg.autoAngles/Axis` written once per frame by `CG_AddPacketEntities`.
- `cent->lerpOrigin/lerpAngles` written per-entity by `CG_CalcEntityLerpPositions` (on-the-fly, not persisted across frames).
- `cent->miscTime` (sound timing, item respawn) read/written by type handlers.

## Learning Notes

**Idiomatic to Quake III:**
- Extensive use of **trajectory evaluation** (`BG_EvaluateTrajectory`) for both client prediction and entity position extrapolation. Modern engines typically use explicit velocity/position updates.
- **Global state struct pattern** (`cg`, `cgs`) avoids deep pointer chains; reduces parameter passing but couples subsystems.
- **Flat entity dispatch** (switch on type) instead of virtual methods, reflecting constraints of the QVM VM environment circa 1999.
- **BSP brush model special-casing** for sound positioning (use geometric midpoint) reflects tight coupling to BSP level structure.

**Modern engine equivalents:**
- This role would typically be split: snapshot processing → predicted state, rendering → scene graph/entity component system.
- Interpolation might use fixed timesteps (render frames decoupled from tick rate) rather than per-frame fractions.
- Hierarchical attachment via scene graph nodes instead of tag queries.
- Event dispatch via message queues or observer patterns rather than polling `eventSequence`.

**ECS lesson:**
- The file demonstrates why ECS systems exist: type-specific rendering logic (players have skeletal animation, missiles have trails) is scattered across multiple handlers. A data-driven approach could reduce branching.

## Potential Issues

1. **const-casting in tag attachment** (`CG_PositionEntityOnTag`): Violates const semantics to satisfy `MatrixMultiply`'s API. Would benefit from const-correct matrix library.

2. **Per-frame lerpOrigin mutation** (`CG_Item`, `CG_Missile`): Bobbing offset and mover adjustment directly modify `cent->lerpOrigin`. If a bug causes double-application, it's easy to miss. Comments indicating "fresh computation" would help.

3. **No bounds checking on modelindex** (`CG_Item`): Comment says "Bad item index %i on entity" after `CG_Error`, but `CG_General` silently returns if `!modelindex`. Inconsistent error handling.

4. **FIXME: angular mover correction** (`CG_AdjustPositionForMover`): Positions on rotating platforms won't include rotation offset. May cause visual misalignment during rotational motion.

5. **Speaker entity relies on clientNum field**: Comment says `// FIXME: use something other than clientNum...`. Fragile repurposing of state fields for unrelated logic.

6. **Auto-rotation global recomputation every frame**: Cheap (single sin/cos per frame), but if item count grows or frame budget tightens, could be optimized with delta-based angular accumulation.
