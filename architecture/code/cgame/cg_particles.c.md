# code/cgame/cg_particles.c

## File Purpose
Implements a software particle system for the cgame module, managing a fixed pool of particles that simulate weather (snow, flurry, bubbles), combat effects (blood, smoke, sparks, explosions), and environmental effects (oil slicks, dust). Particles are submitted each frame as raw polygons to the renderer via `trap_R_AddPolyToScene`.

## Core Responsibilities
- Maintain a free-list / active-list pool of `MAX_PARTICLES` (8192) particles
- Initialize and register animated shader sequences used by explosion/anim particles
- Classify particles by type and build camera-aligned or flat polygon geometry each frame
- Apply simple physics (position = origin + vel*t + accel*t²) during the update pass
- Cull expired particles back to the free list; cull distant particles to avoid poly budget overruns
- Provide typed spawn helpers called from other cgame subsystems (weapons, events, entities)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cparticle_t` | struct | Per-particle state: position, velocity, acceleration, alpha, size, type, shader, roll, animation index |
| `particle_type_t` | enum | Discriminates rendering/update path (P_WEATHER, P_SMOKE, P_ANIM, P_BLEED, P_SPRITE, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `particles[MAX_PARTICLES]` | `cparticle_t[8192]` | global | Fixed particle pool |
| `active_particles` | `cparticle_t *` | global | Head of active linked list |
| `free_particles` | `cparticle_t *` | global | Head of free linked list |
| `cl_numparticles` | `int` | global | Effective pool size (= MAX_PARTICLES) |
| `initparticles` | `qboolean` | global | Lazy-init guard |
| `vforward/vright/vup` | `vec3_t` | global | View-aligned basis vectors (updated each frame) |
| `rforward/rright/rup` | `vec3_t` | global | Roll-adjusted basis vectors for smoke rotation |
| `oldtime` | `float` | global | Previous frame time for roll accumulation |
| `roll` | `float` | file-static | Cumulative roll angle for smoke particles |
| `shaderAnims[32][64]` | `qhandle_t` | file-static | Preloaded animated shader frames |
| `shaderAnimNames/Counts/STRatio` | arrays | file-static | Animation metadata (name, frame count, aspect ratio) |
| `numShaderAnims` | `int` | file-static | Count of registered animations |

## Key Functions / Methods

### CG_ClearParticles
- Signature: `void CG_ClearParticles(void)`
- Purpose: Reset pool to all-free, register all animated shader sequences.
- Inputs: None (reads `shaderAnimNames`, `shaderAnimCounts`).
- Outputs/Return: void
- Side effects: Zeroes `particles[]`, rebuilds free list, populates `shaderAnims[][]`, sets `initparticles = qtrue`, resets `oldtime`.
- Calls: `trap_R_RegisterShader`, `va`

### CG_AddParticleToScene
- Signature: `void CG_AddParticleToScene(cparticle_t *p, vec3_t org, float alpha)`
- Purpose: Build the polygon geometry for a single particle based on its type, then submit it to the renderer.
- Inputs: Particle state pointer, current world position, current alpha.
- Outputs/Return: void (submits poly to renderer or returns early on cull).
- Side effects: May reset `p->time`/`p->org` for looping weather particles; calls `trap_R_AddPolyToScene`. Mutates `p->pshader` for P_ANIM (selects frame). Mutates `p->accumroll` for P_SMOKE rotation.
- Calls: `Distance`, `VectorMA`, `VectorCopy`, `VectorSet`, `vectoangles`, `AngleVectors`, `trap_R_AddPolyToScene`, `crandom`, `floor`, `sin`, `cos`, `DEG2RAD`
- Notes: Distance culls weather/smoke-impact particles beyond 1024 units. P_ANIM particles within `width/1.5` of the player are skipped (avoid inside-sprite artifact). P_FLAT geometry is laid in the XY plane regardless of view direction.

### CG_AddParticles
- Signature: `void CG_AddParticles(void)`
- Purpose: Per-frame update: advance particle physics, expire dead particles, rebuild active list, call `CG_AddParticleToScene` for each live particle.
- Inputs: None (reads `cg.time`, `cg.refdef`).
- Outputs/Return: void
- Side effects: Modifies `active_particles`, `free_particles`; updates global basis vectors and `roll`/`oldtime`; calls renderer indirectly.
- Calls: `CG_ClearParticles` (lazy init), `VectorCopy`, `vectoangles`, `AngleVectors`, `CG_AddParticleToScene`
- Notes: Physics uses `t = (cg.time - p->time) * 0.001`; quadratic integration. Particles with `alpha <= 0` are freed. Some types (P_SMOKE, P_ANIM, P_BLEED, P_FLAT_SCALEUP_FADE, P_WEATHER_FLURRY) also expire on `endtime`. P_BAT/P_SPRITE with `endtime < 0` are one-shot temporaries.

### Spawn helpers (summary)
- `CG_ParticleSnow` / `CG_ParticleBubble` — weather volume particles, PVS-linked via `snum`/`link`.
- `CG_ParticleSnowFlurry` — directional flurry from a centity.
- `CG_ParticleSmoke` — rising smoke column from a centity.
- `CG_ParticleBulletDebris` / `CG_ParticleSparks` — small tracer-shader debris trails.
- `CG_ParticleExplosion` — P_ANIM particle looked up by animation string name.
- `CG_ParticleImpactSmokePuff` — rotating smoke puff at impact point.
- `CG_Particle_Bleed` / `CG_ParticleBloodCloud` — blood effect on hit/gib.
- `CG_BloodPool` — flat scaled-up blood decal validated by `ValidBloodPool` trace.
- `CG_Particle_OilParticle` / `CG_Particle_OilSlick` / `CG_OilSlickRemove` — oil spill effect lifecycle.
- `CG_ParticleDust` / `CG_ParticleMisc` — generic dust column and sprite particle.
- `CG_SnowLink` — enable/disable PVS-linked snow particles by entity frame id.
- `CG_NewParticleArea` — parses a config string to bulk-spawn snow or bubble particles.
- `CG_AddParticleShrapnel` — stub (returns immediately).

## Control Flow Notes
- `CG_ClearParticles` is called at level load (from cgame init) and lazily from `CG_AddParticles`.
- `CG_AddParticles` is called once per rendered frame from `CG_DrawActiveFrame` (cg_view.c), after scene setup and before `trap_R_RenderScene`.
- Spawn helpers are called from event handlers (`cg_event.c`, `cg_weapons.c`) and entity processors (`cg_ents.c`) in response to game events.

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `cg` (global `cg_t`), `cgs` (global `cgs_t`) — read for time, refdef, snapshot, media handles, GL config
- `trap_R_RegisterShader`, `trap_R_AddPolyToScene` — renderer syscalls (defined in cgame syscall layer)
- `trap_CM_BoxTrace` (via `CG_Trace`) — used in `ValidBloodPool`
- `crandom`, `random`, `VectorMA`, `VectorCopy`, `vectoangles`, `AngleVectors`, `Distance`, `VectorLength`, `VectorNegate`, `VectorClear`, `VectorSet`, `DEG2RAD` — defined in `q_shared`/`q_math`
- `COM_Parse`, `stricmp`, `atoi`, `atof`, `va`, `memset` — standard/engine string utilities
- `CG_ConfigString`, `CG_Printf`, `CG_Error`, `CG_Trace` — defined elsewhere in cgame
