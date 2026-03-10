# code/renderer/tr_scene.c — Enhanced Analysis

## Architectural Role

`tr_scene.c` is the **renderer's public submission API** — the sole legal entry point for all game-side content into the rendering pipeline. It sits at the exact boundary between the cgame VM (which submits scene objects via `trap_R_*` syscalls) and the renderer's internal front-end (`tr_main.c`, `tr_world.c`). All seven `RE_*` functions defined here are registered directly into the `refexport_t` vtable in `tr_init.c` and are dispatched by `cl_cgame.c` when servicing cgame VM syscalls. Structurally, this file is the left wall of the renderer's double-buffered SMP pipeline: it writes into `backEndData[tr.smpFrame]`, while `tr_backend.c` reads from the previous frame's buffer on a separate thread.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/client/cl_cgame.c`** — `CL_CgameSystemCalls` dispatches cgame `trap_R_AddRefEntityToScene`, `trap_R_AddPolyToScene`, `trap_R_AddLightToScene`, `trap_R_AddAdditiveLightToScene`, `trap_R_RenderScene`, `trap_R_ClearScene` directly to the `RE_*` functions via the `refexport_t re` vtable.
- **`code/client/cl_ui.c`** — same vtable path; UI VM's `trap_R_RenderScene` drives player model previews in menus.
- **`code/renderer/tr_init.c`** — populates `refexport_t` with all `RE_*` function pointers returned by `GetRefAPI`; this is the only place these functions are formally registered.
- **`code/renderer/tr_main.c`** — calls `R_AddPolygonSurfaces` from inside `R_RenderView` as part of building the draw-surface sort list; also uses `r_firstSceneDrawSurf` to know the current surface count baseline.
- **`code/renderer/tr_cmds.c`** or platform frame-start path — calls `R_ToggleSmpFrame` once per frame to advance the double-buffer.

### Outgoing (what this file depends on)

- **`tr_main.c`** — calls `R_RenderView`, `R_AddDrawSurf`, `AddPointToBounds`; these constitute the core of the front-end rendering pipeline.
- **`tr_shader.c`** — calls `R_GetShaderByHandle` in `R_AddPolygonSurfaces` to resolve the shader from a client-provided `qhandle_t`.
- **`tr_local.h` globals** — reads/writes `tr` (`trGlobals_t`): `tr.smpFrame`, `tr.refdef`, `tr.world`, `tr.registered`, `tr.frameSceneNum`, `tr.sceneCount`, `tr.frontEndMsec`, `tr.currentEntityNum`, `tr.shiftedEntityNum`.
- **`backEndData[]`** — directly indexes the double-buffered arrays: `entities`, `dlights`, `polys`, `polyVerts`, `drawSurfs`, `commands`.
- **`glConfig`** — reads `hardwareType` to gate RIVA128, Permedia2, and Rage Pro hardware workarounds.
- **`ri` (`refimport_t`)** — calls `ri.Printf`, `ri.Error`, `ri.Milliseconds`; this is the renderer's only permitted back-channel to `qcommon`.
- **CVars** `r_smp`, `r_norefresh`, `r_dynamiclight`, `r_vertexLight` — read at scene dispatch time.

## Design Patterns & Rationale

**Double-buffered producer/consumer (SMP):** The `backEndData[tr.smpFrame]` double-buffer is the core SMP idiom. The front-end (this file + `tr_main.c`) writes frame N into buffer `smpFrame`; the back-end (`tr_backend.c`) simultaneously reads frame N-1 from the other buffer. `R_ToggleSmpFrame` XOR-flips the index. When SMP is off, `smpFrame` is always 0 and there's no threading overhead.

**Multi-scene partitioning via offset pairs:** Each `RE_ClearScene` + `RE_RenderScene` pair forms a logical scene. The `r_firstScene*` / `r_num*` counter pairs slice the flat arrays so that each scene gets an independent sub-range. This avoids per-scene allocation and supports multiple 3D renders in one frame (game world, HUD weapon, menu player model) at near-zero overhead. The same pattern repeats across entities, dlights, polys, and draw surfaces.

**Hardware workarounds as first-class conditionals:** Rather than abstracting broken GPU behavior, Quake III embeds explicit `glConfig.hardwareType` checks directly (Rage Pro forced white vertex colors; Riva128/Permedia2 skip dlights entirely). This is a pragmatic era-specific tradeoff: the hardware list was finite and known at ship time.

**Fog assignment at submission time:** Fog classification for polys happens in `RE_AddPolyToScene` (front-end) rather than at draw time (back-end). This trades submission-time AABB iteration against per-draw branching — a tradeoff that makes the back-end simpler.

## Data Flow Through This File

```
cgame VM (trap_R_Add*)
        │
        ▼
RE_AddRefEntityToScene   ──► backEndData[smpFrame]->entities[r_numentities++]
RE_AddPolyToScene        ──► backEndData[smpFrame]->polys[r_numpolys++]
                              polyVerts[r_numpolyverts++]
                              + fog AABB test against tr.world->fogs[]
RE_AddDynamicLightToScene ──► backEndData[smpFrame]->dlights[r_numdlights++]
        │
        ▼  (RE_RenderScene called)
tr.refdef ◄── sliced views into backEndData arrays (entity[firstScene..num], etc.)
        │
        ▼
R_RenderView(&parms)   [tr_main.c]
        │
        ├─► R_AddPolygonSurfaces() ── reads tr.refdef.polys ──► R_AddDrawSurf()
        ├─► BSP traversal / entity add / dlight distribution
        └─► back-end command queue ──► tr_backend.c (possibly on second thread)
        │
        ▼
r_firstScene* offsets advanced → next scene starts cleanly
```

Key state transition: `tr.refdef` is the bridge — it is populated from the external `refdef_t` (client coordinate space) and from the accumulated `backEndData` slices before being handed to `R_RenderView`.

## Learning Notes

**The `RE_` / `R_` naming convention** is intentional: `RE_` functions are public renderer exports (appear in `refexport_t`); `R_` functions are internal renderer calls. This prefix discipline acts as a poor-man's access control in C.

**`lightingCalculated = qfalse` on entity submission** illustrates deferred computation: entity lighting is expensive and is computed lazily during the front-end traversal in `tr_light.c`, not at submission time.

**`tr.frameSceneNum` vs `tr.sceneCount`:** Both are incremented in `RE_RenderScene`. `tr.sceneCount` is an absolute monotonic counter used by the lens flare visibility system (`tr_flares.c`) to distinguish which scene a surface was last visible in — without this, flares from the game view would bleed into HUD sub-scenes.

**No scene graph or ECS:** Quake III uses a flat array of submitted objects with a single-pass sort (shader/entity/fog/dlight bits packed into a 64-bit key in `tr_main.c`). Modern engines use explicit scene graphs, spatial hierarchies, or ECS component queries; Q3's approach is simpler and cache-friendlier for its entity counts but doesn't scale to large open worlds.

**Y-axis flip in `RE_RenderScene`:** `parms.viewportY = glConfig.vidHeight - (tr.refdef.y + tr.refdef.height)` converts from the refdef's top-origin convention (shared with the rest of the engine/cgame) to OpenGL's bottom-origin convention. This conversion is localized here so callers never need to know about GL's coordinate system.

## Potential Issues

- **`r_numpolys`/`r_numpolyverts` silent drop:** When the poly buffer is full, `RE_AddPolyToScene` silently discards all remaining polys for that call (even partially: it returns from the middle of the `j` loop). A high-volume caller submitting N polys will lose polys N+1 through the end without any per-caller feedback.
- **Fog test correctness:** The AABB overlap test assigns `fogIndex = 0` (no fog) if the poly straddles no fog volume exactly. Polys partially intersecting a fog boundary get fog only if the bounding box overlaps — vertex-level fog blending is not considered at submission time.
- **`r_firstSceneDrawSurf` initialization asymmetry:** `R_ToggleSmpFrame` resets it to 0, but `RE_RenderScene` sets it to `tr.refdef.numDrawSurfs` post-render. If `RE_RenderScene` is called without a preceding `RE_ClearScene`, the entity/dlight/poly slices are still correct (they derive from `r_firstScene*` set at `RE_ClearScene` time), but this relies on call-site discipline.
