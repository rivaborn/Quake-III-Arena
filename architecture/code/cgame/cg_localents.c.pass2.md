# code/cgame/cg_localents.c — Enhanced Analysis

## Architectural Role

This file implements the **client-side local entity pool and frame-by-frame renderer submission pipeline** within the cgame VM. It is a pure client-side subsystem: local entities are spawned by event handlers (gibs, explosions, score plums, particle effects) and are never synchronized with the server. Each frame, `CG_AddLocalEntities` (called from `cg_view.c`'s `CG_DrawActiveFrame`) iterates all active entities, applies per-type transformations, and submits commands to the **Renderer** subsystem via `trap_R_AddRefEntityToScene`. This decouples transient visual effects from the authoritative server state and allows the client to synthesize rich feedback (blood trails, impact marks, floating scores) without server involvement.

## Key Cross-References

### Incoming (who depends on this file)
- **`cg_view.c`**: `CG_DrawActiveFrame` calls `CG_AddLocalEntities` once per rendered frame to update and render all active local entities
- **`cg_effects.c`, `cg_marks.c`, `cg_weapons.c`**: Event handlers call `CG_AllocLocalEntity` to spawn gibs, smoke puffs, explosions, and other transient effects during event processing (`EV_*` dispatch in cgame)
- **Server snapshot processing**: Indirectly, via cgame's `CG_ProcessSnapshots`; entity/event changes trigger local-entity spawns
- **`cg_local.h`**: Type definitions and extern declarations consumed by other cgame modules

### Outgoing (what this file depends on)
- **Renderer subsystem**: `trap_R_AddRefEntityToScene` (submit mesh for rendering), `trap_R_AddLightToScene` (submit dynamic light); these are the primary clients of local-entity state
- **Shared physics** (`bg_pmove.c`): `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — shared deterministically between game VM and cgame VM for trajectory prediction
- **Collision/trace** (`cg_predict.c`): `CG_Trace` for fragment→world collision detection
- **Collision model** (`qcommon/cm_*.c` via syscalls): `trap_CM_PointContents` for nodrop zone detection
- **Sound system**: `trap_S_StartSound`, `trap_S_StartLocalSound` for impact/bounce sounds (via `CG_FragmentBounceSound`)
- **Effects/marks pipeline** (`cg_effects.c`, `cg_marks.c`): `CG_SmokePuff`, `CG_ImpactMark`, `CG_GibPlayer` spawn additional entities
- **Global state** (`cg_main.c`): `cg`, `cgs` read throughout (time, frame timing, shader/sound media handles, refdef, PVS context)

## Design Patterns & Rationale

**Fixed pool + LRU eviction**: 512 `localEntity_t` slots allocated statically; a doubly-linked active list and singly-linked free list avoid per-frame malloc/free overhead. When exhausted, `CG_AllocLocalEntity` evicts the oldest active entity. This is **idiomatic for client-side effects**: visual drops are acceptable; deterministic behavior under load matters more than perfect fidelity.

**Per-type dispatch via switch statement**: `CG_AddLocalEntities` iterates backwards (so newly spawned mid-frame effects appear this frame) and dispatches each entity to a type-specific update function (`CG_AddFragment`, `CG_AddExplosion`, `CG_AddScaleFade`, etc.). This is a straightforward **visitor pattern**; alternatives (vtable per entity, inheritance hierarchy) would be heavier in a C codebase.

**Shared trajectory evaluation** with game VM: Both game and cgame link `bg_pmove.c`'s `BG_EvaluateTrajectory`, ensuring visual consistency. Fragment physics on the client matches the authoritative server simulation (within rounding).

**Asymmetric list design**: Active list is doubly-linked (O(1) removal from middle); free list is singly-linked (stack-like, O(1) pop). This trades storage for iteration convenience and is typical of memory-pool designs.

## Data Flow Through This File

**Initialization** → `CG_InitLocalEntities` (called at cgame startup and tournament restart) zeroes the pool and chains all 512 slots into the free list.

**Per-frame input** → Event handlers (sound events, gibs, weapon impact) call `CG_AllocLocalEntity` to pop a slot from the free list, zero it, and link it into the active list. Trajectories, shaders, colors, and lifetime parameters are set by the spawner.

**Per-frame update** → `CG_AddLocalEntities` is called after entity and mark processing. It walks the active list *backwards* (critical: new effects added this frame appear in the same frame's render), checks expiration (`le->endTime <= cg.time`), and dispatches:
- **Physics fragments** (`LE_FRAGMENT`): trace movement, collide, bounce, reflect, emit blood trails, leave marks, play sounds
- **Stationary fragments** (`TR_STATIONARY`): sink into ground over `SINK_TIME`, then free
- **Trivial effects** (`LE_MOVE_SCALE_FADE`, `LE_SCALE_FADE`, `LE_FALL_SCALE_FADE`, `LE_FADE_RGB`): evaluate trajectory, fade alpha/scale, cull if camera is inside, submit
- **Score plums** (`LE_SCOREPLUM`): drift laterally with sinusoid, fade, color-code by magnitude
- **Explosions** (`LE_EXPLOSION`, `LE_SPRITE_EXPLOSION`): render geometry + dynamic light that fades over lifetime

**Per-frame output** → Each type calls `trap_R_AddRefEntityToScene` (or `trap_R_AddLightToScene` for lights), submitting to the renderer's command buffer. The renderer sorts and batches these commands in the back-end.

**Cleanup** → Expired or invalid entities (nodrop zones, camera-inside culling) call `CG_FreeLocalEntity`, moving them back to the free list for reuse.

## Learning Notes

**Client-side simulation decoupling**: Unlike the authoritative server, the client can emit transient visual-only effects without network sync. This was a design win in early 2000s netcode: rich client feedback with minimal bandwidth.

**Pool-based memory in the QVM era**: Q3A predates modern garbage collection and dynamic allocation in game engines. The fixed 512-entity pool + LRU eviction is a pre-modern pattern, but it was optimal given console port constraints (PS2, Xbox had tight VRAM budgets) and VM execution overhead. Modern engines use dynamic arrays or object pools with growth.

**Shared physics across two VMs**: Both game and cgame compile `bg_pmove.c` to ensure `BG_EvaluateTrajectory` is identical. Fragment trajectories on the client (visual) and on the server (authoritative) follow the same math—a subtle but critical design decision for deterministic replay and cheat prevention.

**Coupled renderer + sound systems**: `CG_AddFragment` does not just submit visuals; it also plays sounds (`CG_FragmentBounceSound`) and leaves decal marks (`CG_FragmentBounceMark`). This tight coupling is pragmatic but reflects the era's "all-in-one" game loop. Modern engines separate rendering, audio, and decal systems.

**Backward iteration for same-frame spawns**: `CG_AddLocalEntities` walks the active list backwards so effects spawned mid-frame (e.g., blood trails from a gib, marks from a bounce) are processed immediately. This ensures visual correctness without deferred rendering.

## Potential Issues

**Silent pool exhaustion**: Under high load (e.g., many simultaneous explosions or gore-heavy gameplay), the 512-entity pool may overflow silently, evicting the oldest effect. On low-end hardware of the era, this could cause loss of visual feedback, but it guaranteed no memory allocation failure or hang.

**Light/dlight coupling**: `CG_AddExplosion` directly calls `trap_R_AddLightToScene`. The light's color and intensity are baked into the local entity struct rather than computed dynamically. If the renderer's light system changes, this code path becomes fragile.

**No thread safety**: The global `cg_localEntities`, `cg_activeLocalEntities`, and `cg_freeLocalEntities` are not protected by locks. The engine's SMP mode (front-end/back-end split in `renderer/tr_backend.c`) does not call cgame in parallel, so this is safe in practice, but it's worth noting.
