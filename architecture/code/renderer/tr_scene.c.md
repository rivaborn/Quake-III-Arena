# code/renderer/tr_scene.c

## File Purpose
Implements the renderer's scene submission API, acting as the front-end interface between the game/cgame modules and the renderer pipeline. It accumulates entities, dynamic lights, and polygons into double-buffered back-end data arrays, then triggers a view render pass via `RE_RenderScene`.

## Core Responsibilities
- Toggle SMP (symmetric multi-processing) double-buffer frames and reset scene counters
- Accept and buffer `refEntity_t` submissions into `backEndData` entity arrays
- Accept and buffer dynamic light submissions (normal and additive) into `backEndData` dlight arrays
- Accept and buffer client-submitted polygons (`srfPoly_t`) into `backEndData` poly/polyVert arrays, including fog volume assignment
- Flush all buffered polygon surfaces into the current view's draw surface list
- Populate `tr.refdef` from the `refdef_t` descriptor and invoke `R_RenderView` to execute a 3D render pass
- Support multiple scenes per frame (3D game view, HUD models, menus) via `firstScene*` offset bookkeeping

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `srfPoly_t` | struct | A client-submitted polygon surface with shader handle, fog index, and vertex pointer |
| `viewParms_t` | struct | Per-view render parameters: viewport, FOV, orientation, portal flag |
| `trRefdef_t` | struct | Extended refdef held in `tr.refdef`; includes entity/dlight/poly/drawsurf slices |
| `dlight_t` | struct | Dynamic light: origin, color, radius, additive flag |
| `trRefEntity_t` | struct | Extended entity: wraps `refEntity_t` plus lighting cache fields |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `r_firstSceneDrawSurf` | `int` | global | Start index of draw surfaces for the current scene within the frame |
| `r_numdlights` | `int` | global | Running count of dlights submitted this frame |
| `r_firstSceneDlight` | `int` | global | Start index of dlights for the current scene |
| `r_numentities` | `int` | global | Running count of entities submitted this frame |
| `r_firstSceneEntity` | `int` | global | Start index of entities for the current scene |
| `r_numpolys` | `int` | global | Running count of polys submitted this frame |
| `r_firstScenePoly` | `int` | global | Start index of polys for the current scene |
| `r_numpolyverts` | `int` | global | Running count of poly vertices submitted this frame |

## Key Functions / Methods

### R_ToggleSmpFrame
- **Signature:** `void R_ToggleSmpFrame( void )`
- **Purpose:** Advances the SMP double-buffer index and resets all scene counters for a new frame.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Flips `tr.smpFrame` (XOR with 1 if SMP enabled), zeroes `backEndData[tr.smpFrame]->commands.used`, resets all `r_firstScene*` and `r_num*` counters.
- **Calls:** None
- **Notes:** Must be called once per frame before any scene submissions. SMP disabled forces `smpFrame = 0`.

---

### RE_ClearScene
- **Signature:** `void RE_ClearScene( void )`
- **Purpose:** Marks the start of a new logical scene within the current frame, so subsequent submissions are attributed to this scene only.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `r_firstScene*` offsets to current `r_num*` counts, effectively partitioning the frame's accumulated data.
- **Calls:** None

---

### RE_AddPolyToScene
- **Signature:** `void RE_AddPolyToScene( qhandle_t hShader, int numVerts, const polyVert_t *verts, int numPolys )`
- **Purpose:** Submits one or more client polygons into the current frame's poly buffer, with fog volume classification.
- **Inputs:** Shader handle, vertex count per poly, vertex array, number of polys to add.
- **Outputs/Return:** None (silently drops if buffers full)
- **Side effects:** Writes into `backEndData[tr.smpFrame]->polys` and `->polyVerts`; increments `r_numpolys`, `r_numpolyverts`. On GLHW_RAGEPRO, forces full-white modulate.
- **Calls:** `AddPointToBounds`, `ri.Printf`
- **Notes:** Fog assignment iterates `tr.world->fogs` via AABB overlap test; fogIndex 0 means no fog. Capacity-overflow drops with `PRINT_DEVELOPER` only.

---

### RE_AddRefEntityToScene
- **Signature:** `void RE_AddRefEntityToScene( const refEntity_t *ent )`
- **Purpose:** Copies a client-provided entity descriptor into the back-end entity array for the current frame.
- **Inputs:** Pointer to `refEntity_t`.
- **Outputs/Return:** None
- **Side effects:** Writes to `backEndData[tr.smpFrame]->entities[r_numentities]`, sets `lightingCalculated = qfalse`, increments `r_numentities`.
- **Calls:** None
- **Notes:** Caps at `ENTITYNUM_WORLD`; validates `reType` range, calling `ri.Error(ERR_DROP)` on bad value.

---

### RE_AddDynamicLightToScene
- **Signature:** `void RE_AddDynamicLightToScene( const vec3_t org, float intensity, float r, float g, float b, int additive )`
- **Purpose:** Submits a dynamic point light into the frame's dlight buffer.
- **Inputs:** World origin, intensity (used as radius), RGB color, additive blend flag.
- **Outputs/Return:** None
- **Side effects:** Writes to `backEndData[tr.smpFrame]->dlights[r_numdlights++]`.
- **Calls:** `VectorCopy`
- **Notes:** Silently rejected for GLHW_RIVA128 and GLHW_PERMEDIA2 (broken blend modes); also rejected if intensity ≤ 0 or `MAX_DLIGHTS` reached.

---

### RE_RenderScene
- **Signature:** `void RE_RenderScene( const refdef_t *fd )`
- **Purpose:** Finalizes `tr.refdef` from a submitted `refdef_t`, builds view parameters, and dispatches a full 3D render pass.
- **Inputs:** `refdef_t` descriptor with viewport, FOV, view origin/axes, time, area mask, flags.
- **Outputs/Return:** None
- **Side effects:** Populates `tr.refdef`; increments `tr.frameSceneNum`, `tr.sceneCount`; calls `R_RenderView`; advances `r_firstScene*` offsets post-render; accumulates `tr.frontEndMsec`.
- **Calls:** `GLimp_LogComment`, `ri.Milliseconds`, `Com_Memcpy`, `VectorCopy`, `Com_Memset`, `R_RenderView`, `ri.Error`
- **Notes:** Converts Y-coordinate from top-origin (refdef) to bottom-origin (GL). Disables dlights globally if `r_dynamiclight == 0`, `r_vertexLight == 1`, or GLHW_PERMEDIA2. Requires a loaded world unless `RDF_NOWORLDMODEL` is set.

---

### R_AddPolygonSurfaces
- **Signature:** `void R_AddPolygonSurfaces( void )`
- **Purpose:** Flushes all buffered polygons for the current scene into the active view's draw surface list.
- **Inputs:** None (reads `tr.refdef.polys` / `numPolys`)
- **Outputs/Return:** None
- **Side effects:** Calls `R_AddDrawSurf` for each poly; sets `tr.currentEntityNum = ENTITYNUM_WORLD`.
- **Calls:** `R_GetShaderByHandle`, `R_AddDrawSurf`

## Control Flow Notes
- Called once per frame: `R_ToggleSmpFrame` → `RE_ClearScene` → N × `RE_AddRefEntityToScene` / `RE_AddPolyToScene` / `RE_AddDynamicLightToScene` → `RE_RenderScene`.
- `RE_RenderScene` internally calls `R_RenderView`, which calls `R_AddPolygonSurfaces` as part of building the draw surface list before dispatching to the back end.
- Multiple scenes per frame are supported by the `r_firstScene*` partitioning idiom; each `RE_RenderScene` call advances the offsets so the next scene starts cleanly.

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:** `tr` (`trGlobals_t`), `backEndData` (`backEndData_t*[SMP_FRAMES]`), `glConfig` (`glconfig_t`), `ri` (`refimport_t`), `r_smp`, `r_norefresh`, `r_dynamiclight`, `r_vertexLight` (cvars), `max_polys`, `max_polyverts`, `R_RenderView`, `R_AddDrawSurf`, `R_GetShaderByHandle`, `AddPointToBounds`, `GLimp_LogComment`
