# code/cgame/cg_marks.c

## File Purpose
Manages persistent wall mark decals (bullet holes, burn marks, blood splats) and a full particle simulation system for the cgame module. Despite the filename, the file contains two logically separate systems: mark polys and a Ridah-era particle engine that was folded in.

## Core Responsibilities
- Maintain a fixed-size pool of `markPoly_t` nodes using a doubly-linked active list and singly-linked free list
- Project impact decals onto world geometry by clipping a quad against BSP surfaces via `trap_CM_MarkFragments`
- Fade and expire persistent mark polys each frame, submitting survivors to the renderer
- Maintain a fixed-size pool of `cparticle_t` particles (weather, smoke, blood, bubbles, sprites, animations)
- Update and submit particles each frame with physics integration (velocity + acceleration)
- Provide factory functions for spawning typed particles (snow, smoke, sparks, blood, explosions, etc.)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `markPoly_t` | struct (typedef) | One persistent wall decal: linked-list pointers, birth time, shader, alpha-fade flag, color, poly descriptor, and up to `MAX_VERTS_ON_POLY` verts |
| `cparticle_t` | struct (typedef, local) | One particle: physics state (org/vel/accel), visual state (alpha, color, width/height), type, shader, animation data |
| `particle_type_t` | enum (local) | Discriminates particle rendering path: `P_WEATHER`, `P_SMOKE`, `P_BLEED`, `P_ANIM`, `P_SPRITE`, `P_BUBBLE`, etc. |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cg_activeMarkPolys` | `markPoly_t` | global | Sentinel head of doubly-linked active mark list |
| `cg_freeMarkPolys` | `markPoly_t *` | global | Head of singly-linked free mark list |
| `cg_markPolys[MAX_MARK_POLYS]` | `markPoly_t[256]` | global | Fixed storage for all mark polys |
| `markTotal` | `int` | static (file) | Running count of marks ever allocated (unused for eviction logic) |
| `particles[MAX_PARTICLES]` | `cparticle_t[1024]` | global | Fixed particle pool |
| `active_particles` / `free_particles` | `cparticle_t *` | global | Heads of active/free particle singly-linked lists |
| `initparticles` | `qboolean` | global | Lazy-init flag; triggers `CG_ClearParticles` on first `CG_AddParticles` call |
| `pvforward/pvright/pvup` | `vec3_t` | global | View-axis vectors for billboard alignment (set each frame) |
| `rforward/rright/rup` | `vec3_t` | global | Roll-adjusted view vectors for rotating smoke |
| `oldtime` | `float` | global | Previous frame time for roll accumulation |
| `roll` | `float` | static | Accumulated roll angle for smoke rotation |
| `shaderAnims` / `shaderAnimNames` / `shaderAnimCounts` | arrays | static | Registered animation frame shaders (currently only `"explode1"`, 23 frames) |

## Key Functions / Methods

### CG_InitMarkPolys
- Signature: `void CG_InitMarkPolys(void)`
- Purpose: Initializes the mark poly pool; called at startup and tournament restart.
- Inputs: None
- Outputs/Return: None
- Side effects: Zeros `cg_markPolys`, sets up sentinel active list, links all nodes into the free list.
- Calls: `memset`
- Notes: Safe to call repeatedly; fully reinitializes pool.

### CG_AllocMark
- Signature: `markPoly_t *CG_AllocMark(void)`
- Purpose: Allocates one mark poly, evicting the oldest same-timestamp group if the pool is exhausted.
- Inputs: None
- Outputs/Return: Pointer to zeroed, active-list-linked `markPoly_t`.
- Side effects: May call `CG_FreeMarkPoly` multiple times; mutates active/free lists.
- Calls: `CG_FreeMarkPoly`, `memset`
- Notes: Eviction loop removes all marks sharing the oldest `time` value, not just one — prevents partial group orphaning.

### CG_ImpactMark
- Signature: `void CG_ImpactMark(qhandle_t markShader, const vec3_t origin, const vec3_t dir, float orientation, float red, float green, float blue, float alpha, qboolean alphaFade, float radius, qboolean temporary)`
- Purpose: Creates a decal at a world impact point, clipping it against BSP surfaces.
- Inputs: Shader, world origin/normal, rotation angle, RGBA, fade mode, decal radius, temporary flag.
- Outputs/Return: None
- Side effects: Allocates mark polys via `CG_AllocMark`; calls `trap_R_AddPolyToScene` immediately for temporary marks.
- Calls: `VectorNormalize2`, `PerpendicularVector`, `RotatePointAroundVector`, `CrossProduct`, `VectorScale`, `trap_CM_MarkFragments`, `CG_AllocMark`, `trap_R_AddPolyToScene`, `memcpy`
- Notes: Temporary marks (e.g., player shadows) bypass the pool entirely. `cg_addMarks` cvar gates all output.

### CG_AddMarks
- Signature: `void CG_AddMarks(void)`
- Purpose: Per-frame update: expires old marks, fades energy/alpha marks, submits survivors to renderer.
- Inputs: None (reads `cg.time`, `cgs.media.energyMarkShader`)
- Outputs/Return: None
- Side effects: Frees expired marks; modifies `verts[j].modulate` in-place; calls `trap_R_AddPolyToScene`.
- Calls: `CG_FreeMarkPoly`, `trap_R_AddPolyToScene`
- Notes: `MARK_TOTAL_TIME` = 10 s lifetime; `MARK_FADE_TIME` = last 1 s. Energy marks use RGB fade; `alphaFade` marks use alpha channel fade; others use RGB fade.

### CG_AddParticles
- Signature: `void CG_AddParticles(void)`
- Purpose: Per-frame particle simulation: integrates physics, expires dead particles, submits live ones.
- Inputs: None (reads `cg.time`, `cg.refdef`)
- Outputs/Return: None
- Side effects: Updates `active_particles`, `free_particles`; computes view vectors; calls `CG_AddParticleToScene`; updates `roll`, `oldtime`.
- Calls: `CG_ClearParticles`, `VectorCopy`, `vectoangles`, `AngleVectors`, `CG_AddParticleToScene`
- Notes: Physics: `org = p->org + vel*t + accel*t²`. Alpha <= 0 recycles particle. Temporary sprites (`endtime < 0`) render once then free.

### CG_AddParticleToScene
- Signature: `void CG_AddParticleToScene(cparticle_t *p, vec3_t org, float alpha)`
- Purpose: Constructs poly vertices for a single particle based on its type and submits to renderer.
- Inputs: Particle, evaluated world position, current alpha.
- Outputs/Return: None
- Side effects: Calls `trap_R_AddPolyToScene` (3 verts for weather triangles, 4 for quads).
- Calls: `Distance`, `VectorMA`, `VectorCopy`, `vectoangles`, `AngleVectors`, `trap_R_AddPolyToScene`
- Notes: Distance culling at 1024 units for weather/bubble/smoke-impact types. `P_ANIM` selects shader frame by ratio into `shaderAnims`. `P_FLAT_SCALEUP` uses sinusoidal rotation.

## Control Flow Notes
- **Init:** `CG_InitMarkPolys` called from `CG_Init` (via `cg_main.c`). `CG_ClearParticles` is lazy — triggered on first `CG_AddParticles` call if `initparticles == qfalse`.
- **Per-frame:** `CG_AddMarks` and `CG_AddParticles` are called each render frame from `CG_DrawActiveFrame` after scene setup, before `trap_R_RenderScene`.
- **Event-driven:** `CG_ImpactMark` is called from weapon hit handlers (`cg_weapons.c`). Particle spawn functions are called from `cg_effects.c`, `cg_ents.c`, and `cg_weapons.c`.

## External Dependencies
- `cg_local.h` — all cgame types, globals (`cg`, `cgs`), and trap declarations
- `trap_CM_MarkFragments` — BSP polygon clipping (defined in engine)
- `trap_R_AddPolyToScene` — renderer polygon submission (defined in engine)
- `trap_R_RegisterShader` — shader registration for particle anim frames (defined in engine)
- `VectorNormalize2`, `PerpendicularVector`, `RotatePointAroundVector`, `CrossProduct`, `VectorMA`, `DotProduct`, `Distance`, `AngleVectors`, `vectoangles` — math utilities (defined in `q_math.c`/`q_shared.c`)
- `cg_addMarks` cvar — gates mark/particle submission; declared in `cg_main.c`
- `cgs.media.energyMarkShader`, `tracerShader`, `smokePuffShader`, `waterBubbleShader` — preloaded media handles
