# code/cgame/cg_marks.c — Enhanced Analysis

## Architectural Role

This file implements **two independent client-side visual effect pools** for the cgame VM: persistent wall mark decals (bullet holes, scorch marks, blood splatters clipped against BSP geometry) and a standalone particle simulation engine. Both systems bypass server synchronization entirely—they are purely local client effects driven by event triggers and per-frame updates during scene rendering. The mark system interfaces directly with the collision model (`trap_CM_MarkFragments`) to project decals onto arbitrary world geometry; the particle system provides a full physics pipeline for weather, smoke, sparks, and sprite effects. Together they provide the visual "wear and tear" and transient effects that populate a live game scene.

## Key Cross-References

### Incoming (who depends on this file)
- **`CG_InitMarkPolys`**: called from `CG_Init` (cgame module startup, `cg_main.c`)
- **`CG_ImpactMark`**: called from:
  - `cg_weapons.c` (weapon impact hit handlers: bullet, grenade, rocket splash)
  - `cg_effects.c` (generic decal spawning)
  - `cg_ents.c` (entity damage visualization)
- **`CG_AddMarks` / `CG_AddParticles`**: called from `CG_DrawActiveFrame` (cgame per-frame loop, `cg_main.c`), executed after snapshot processing but before `trap_R_RenderScene`
- **`CG_SpawnParticle*` functions** (lines ~1600+): called from:
  - `cg_effects.c` (weapon fire, explosions, environmental effects)
  - `cg_weapons.c` (muzzle flashes, casing eject)
  - `cg_ents.c` (damage blood, death effects, environmental)
- **Global reads**: `cg_addMarks` cvar (gates all output if disabled)

### Outgoing (what this file depends on)
- **Renderer**: `trap_R_AddPolyToScene` (submitted every frame for each mark and particle); `trap_R_RegisterShader` (loads animation frame shaders during `CG_ClearParticles`)
- **Collision model**: `trap_CM_MarkFragments` (BSP quad clipping to project decals onto surfaces)
- **Math utilities** (q_shared.c/q_math.c): `VectorNormalize2`, `PerpendicularVector`, `RotatePointAroundVector`, `CrossProduct`, `VectorMA`, `VectorCopy`, `DotProduct`, `Distance`, `vectoangles`, `AngleVectors`
- **cgame globals**: `cg.time`, `cg.refdef`, `cgs.media.energyMarkShader`, `tracerShader`, `smokePuffShader`, `waterBubbleShader`

## Design Patterns & Rationale

**1. Fixed-Size Pool Allocation (no `malloc`)**
- Both mark and particle systems pre-allocate all nodes (`cg_markPolys[MAX_MARK_POLYS]`, `particles[MAX_PARTICLES]`) at startup
- Rationale: Predictable memory footprint on fixed-RAM console hardware (PS2, Xbox, Dreamcast era); eliminates mid-frame allocation stalls and fragmentation
- Marks use **doubly-linked active list + singly-linked free list**; particles use **singly-linked active/free lists**—both enable O(1) allocation/deallocation

**2. Event-Driven Spawning with Per-Frame Culling**
- `CG_ImpactMark` called event-driven from weapon handlers; `CG_AllocMark` allocates and links
- `CG_AddMarks` and `CG_AddParticles` run every frame, aging and expiring entities
- Rationale: Defers memory contention; amortizes culling cost across frames; allows per-frame fading/animation

**3. Temporary vs. Persistent Marks**
- `temporary=qtrue` marks (shadows) bypass pooling entirely—submitted directly via `trap_R_AddPolyToScene` and forgotten
- `temporary=qfalse` marks are pooled, stored, and re-rendered each frame with fading
- Rationale: Shadows update every frame and don't need persistence; wall decals should accumulate and fade naturally over 10 seconds

**4. Deferred In-Place Color/Alpha Modification**
- Per-frame, `CG_AddMarks` mutates `mark->verts[j].modulate` in-place based on age—either RGB fade (most marks) or alpha-only fade (`alphaFade` flag)
- Energy marks get special RGB fade behavior (fast 450-unit fade then stable)
- Rationale: Avoids allocating separate fade state; leverages polymorphic vertex data layout

## Data Flow Through This File

### Mark Decals
```
Event: Weapon hit → CG_ImpactMark(shader, origin, normal, color, radius)
  ↓ Allocate pool node via CG_AllocMark()
  ↓ Build 4-corner quad in world space around impact origin
  ↓ Call trap_CM_MarkFragments() to clip quad against BSP geometry
  ↓ For each clipped fragment:
      - Convert vertices to UV coords (relative to decal origin/normal)
      - Store in mark->verts[] with RGBA color
  ↓ Store mark: shader, time, alphaFade flag, color
  ↓ (temp marks → trap_R_AddPolyToScene immediately; persistent → linked list)

Per-frame (CG_AddMarks):
  For each active mark:
    - If age > MARK_TOTAL_TIME (10s) → free
    - If age > MARK_TOTAL_TIME - MARK_FADE_TIME (last 1s) → fade modulate
    - If P_ENERGY mark → apply special RGB fade curve
    - trap_R_AddPolyToScene(shader, verts with updated modulate)
```

### Particles
```
Event: Effect trigger → CG_SpawnParticle*(type, org, vel, ...)
  ↓ Allocate from free_particles list
  ↓ Initialize: org, vel, accel, color, shader, type
  ↓ (Lazy init CG_ClearParticles if first call)

Per-frame (CG_AddParticles):
  - Integrate physics: org += vel*dt; vel += accel*dt
  - Update alpha: alpha -= alphavel
  - Recycle if alpha <= 0 or endtime reached
  ↓ Call CG_AddParticleToScene() for each survivor:
      - Distance-cull if > 1024 units away (weather/bubbles/smoke-impact only)
      - Construct 3-4 vertices based on particle type:
        * P_WEATHER, P_BUBBLE: front-facing triangle relative to camera
        * P_SMOKE, P_ANIM, P_ROTATE: billboarded quad with rotation
        * P_FLAT_SCALEUP: scaled quad with sin() rotation animation
      - trap_R_AddPolyToScene(shader or animated frame, verts)
```

## Learning Notes

- **Console-Era Pool Pattern**: This code exemplifies fixed-pool allocation design philosophy of mid-90s game engines (Quake III's targets: PC, PS2, Xbox). No dynamic allocation in hot paths; all sizes predetermined. Modern engines use growing pools or object-component layers, but fixed pools remain relevant for latency-critical, memory-constrained platforms.

- **Shader Animation State Machine**: The `shaderAnims` / `shaderAnimCounts` arrays precompute frame shader handles at init time (`trap_R_RegisterShader` per frame). Frame selection is a simple ratio lookup (`int(frame_index * shaderAnimCounts[i])`), not a dynamic shader transition. This "baked" approach was common before real-time shader compilation.

- **Bilboarded vs. Deferred Rendering**: All particles and marks use immediate-mode submission (`trap_R_AddPolyToScene` per entity per frame). No scene-graph or level-of-detail culling. Contrast with modern engines that batch and cull. The cost was acceptable at 60 FPS with ~1000 particles/marks on 2000-era hardware.

- **Ridah Mark-to-Particle Integration**: The file conflates two unrelated systems because Ridah (author) folded the particle engine into `cg_marks.c` rather than creating `cg_particles.c`. The file comment says "// cg_particles.c" at line ~245, indicating a refactoring artifact or code organization oversight.

- **Deterministic Physics**: Particle physics uses simple **forward Euler integration** (`org += vel * dt`), not constraint-solving or adaptive stepping. For visual-only effects, this determinism (no randomness in integration) allows replay fidelity and offline simulation if needed.

- **No Server Sync**: Unlike networked entities, marks and particles are **purely client-side**. This allows aggressive culling and per-client customization (e.g., blood-disable in competitive mods) without bandwidth cost.

## Potential Issues

1. **Unused `markTotal` Counter**: Incremented in `CG_ImpactMark` but never read; allocation check is commented out. Suggests planned pool saturation warning that was abandoned.

2. **Eviction Policy**: `CG_AllocMark` removes **all** marks with the oldest timestamp, not a single LRU entry. If 50 marks spawn at frame T, the pool exhausts, all 50 are freed together—potentially a visible "pop" of decals vanishing.

3. **Particle Physics Simplicity**: Forward Euler can accumulate error over long simulation times (e.g., smoke drifting 10+ frames). No constraint-solving means particles can tunnel through geometry or drift off-world if unchecked.

4. **Distance Culling Hardcoded at 1024**: Weather and particle effects cull at 1024 units globally. Large maps or long sightlines may show abrupt particle disappearance. Modern engines use view-frustum or fog-based culling.

5. **Mark Vertex Clipping Limit**: `trap_CM_MarkFragments` may return polygons with `>MAX_VERTS_ON_POLY` vertices; code silently clamps (`mf->numPoints = MAX_VERTS_ON_POLY`), potentially distorting large clipped decals. No error or warning.
