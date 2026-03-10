# code/cgame/cg_particles.c — Enhanced Analysis

## Architectural Role

This file implements a **client-side-only soft-body effects layer** that is part of cgame's local entity/effect subsystem (alongside `cg_localents.c` and `cg_marks.c`). Unlike network-synchronized gameplay entities, particles are purely presentational and originate from event handlers (`cg_event.c`), weapon systems (`cg_weapons.c`), and entity processors (`cg_ents.c`). The system feeds raw screen-aligned polygons directly to the renderer's `trap_R_AddPolyToScene` pipeline each frame, bypassing entity slots and PVS culling (except for weather, which uses sector-based `snum` linking).

## Key Cross-References

### Incoming (who depends on this)
- **`cg_view.c:CG_DrawActiveFrame`** — calls `CG_AddParticles()` once per frame after `trap_R_RenderScene` setup
- **`cg_event.c`** — fires `CG_Particle*` spawners on game events (EV_BLOOD, EV_EXPLOSION, EV_SPARKS, etc.)
- **`cg_weapons.c`** — fires `CG_Particle*` spawners for weapon discharge/impact effects
- **`cg_ents.c`** — spawns dust, smoke, flurry particles tied to centity updates
- **`cg_local.h`** — declares public spawn functions as extern for cross-module visibility

### Outgoing (what this depends on)
- **Renderer** — `trap_R_AddPolyToScene(qhandle_t shader, int numverts, const polyVert_t *verts)` to submit raw polys; `trap_R_RegisterShader` for animated frame lookup
- **Collision model** — `trap_CM_BoxTrace` (via `CG_Trace`) used only in `ValidBloodPool` for ground validation
- **Math/utility** — `q_shared.h` (VectorMA, Distance, AngleVectors, crandom)
- **Global state** — `cg` (time, refdef, snap), `cgs` (glconfig, media handles)
- **Config parsing** — `COM_Parse`, `va`, `CG_ConfigString` for `CG_NewParticleArea` bulk spawning

## Design Patterns & Rationale

**Free-list pool pattern**: Preallocates 8192 particles at init; no malloc/free per spawn (allocation happens on level load only). Maintains `active_particles` (linked list of live particles) and `free_particles` (available slots). This avoids frame-time allocation jitter—critical for consistent 60 Hz framerate.

**Type-discriminator enum**: `particle_type_t` (15 variants) controls all branching in both `CG_AddParticleToScene` (rendering) and `CG_AddParticles` (physics/culling). Each type has distinct geometry, culling rules, and physics. Tradeoff: tight coupling between types; adding a new variant requires edits in multiple places.

**Lazy shader animation registration**: Animated shaders are registered on first `CG_ClearParticles` call (usually level load), not on first particle spawn. The `shaderAnims[32][64]` table holds precomputed qhandles. Rationale: registration is expensive; caching amortizes cost across all particles of that type.

**PVS-aware weather rendering**: Particles like snow/bubbles use `snum` (cluster sector number) and `link` (visibility flag) to avoid rendering off-screen weather. Spawned in bulk per sector via `CG_ParticleSnow`, then culled by PVS each frame. Reduces poly count in large outdoor maps.

**Distance-based culling**: P_WEATHER, P_SMOKE_IMPACT particles beyond 1024 units from player are skipped. Protects renderer poly budget in dense effect zones (e.g., village with many fires). Hardcoded constant; not configurable.

**Quadratic physics integration**: Particles advance via `pos = org + vel*t + 0.5*accel*t²` (computed in `CG_AddParticles`). Simple, predictable, and sufficient for 0.5–2 second lifetimes.

## Data Flow Through This File

```
EVENT/WEAPON/ENTITY Handler
    ↓
CG_Particle* (spawn helper) 
    ↓ Allocates from free_particles, links to active_particles
    ↓
CG_AddParticles (each frame)
    ├─ Advance physics (pos, vel, alpha)
    ├─ Cull expired / alpha ≤ 0 → unlink to free_particles
    ├─ For each live particle: CG_AddParticleToScene
    │    ├─ Build quad/tri geometry (screen-aligned or rotated)
    │    ├─ Select shader frame (for P_ANIM)
    │    ├─ Call trap_R_AddPolyToScene → renderer command queue
    │    └─ Return (or early exit on distance/type cull)
    └─ Update basis vectors (vforward/vright/vup, roll accumulator)
    ↓
Renderer
    ├─ Sort all submitted polys by shader
    └─ Render in optimal batch order
```

**Key state transitions**: Free → Active (spawn) → Active (physics/render loop) → Free (expire or alpha ≤ 0).

## Learning Notes

- **Client-side only**: Particles are never mentioned in network protocol or server VM. This is a pure presentation detail, decoupled from gameplay authority.
- **Pool-based object management in C**: This pattern (free-list, linked lists, array-of-structs) is idiomatic to late-1990s/early-2000s game engines before C++ became standard. Modern C++ would use `std::vector` with move semantics or object pools with handles.
- **Shader-driven animation**: Frame selection via `shaderAnims[i][j]` (computed at registration time) is era-typical. Modern engines use GPU-side texture atlases and UV animation in shaders.
- **Screen-aligned billboarding**: Most particle types use `vup/vright` to face the camera. The `P_SPRITE` type additionally supports roll-axis rotation. Avoids 3D mesh overhead.
- **PVS integration in cgame**: Weather uses `snum`/`link` to respect visibility clusters, learned from the server's PVS model. Demonstrates how client-side subsystems can reuse engine spatial concepts.
- **Distance culling as poor-man's LOD**: Hardcoded 1024-unit cull is a simple heuristic to stay under poly budget. Modern engines use hierarchical LOD or GPU frustum culling.

## Potential Issues

- **Type proliferation**: 15 particle types with overlapping semantics (P_SMOKE vs P_SMOKE_IMPACT, P_BUBBLE vs P_BUBBLE_TURBULENT, P_FLAT vs P_FLAT_SCALEUP vs P_FLAT_SCALEUP_FADE). Suggests feature creep; could benefit from parameterization or composition.
- **PVS-linked weather inefficiency**: `CG_ParticleSnow` spawns bulk particles per sector each frame (if `CG_SnowLink` is enabled). Each particle polls `link` in `CG_AddParticleToScene`. No spatial batching or frustum culling; relies on PVS alone.
- **Distance culling is magic-number-based**: Hardcoded `1024` units. Not exposed as a cvar. No per-particle-type or per-effect tuning.
- **No early-out on off-screen quads**: Polygons submitted to the renderer even if fully behind the camera (renderer culls later). Could compute screen-space AABB in cgame for early rejection.
- **Temporal coherence lost**: Each frame rebuilds all vertex data from scratch (VectorMA calls in `CG_AddParticleToScene`). Could optimize by caching world-space corners and only updating when needed (e.g., view rotation, particle motion).
