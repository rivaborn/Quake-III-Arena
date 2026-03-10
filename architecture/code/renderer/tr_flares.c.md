# code/renderer/tr_flares.c

## File Purpose
Implements the light flare rendering subsystem for Quake III Arena's renderer. Flares simulate an ocular effect where bright light sources produce visible glare rings; they use depth buffer readback to determine visibility and interpolate intensity across frames for smooth fading.

## Core Responsibilities
- Maintain a pool of `flare_t` state objects across multiple frames and scenes
- Project 3D flare positions to screen-space coordinates during surface tessellation
- Read back the depth buffer per-flare to test occlusion after opaque geometry is drawn
- Fade flare intensity smoothly in/out using time-based interpolation
- Render each visible flare as a screen-aligned quad in orthographic projection
- Register dynamic light sources (dlights) as flares via `RB_AddDlightFlares`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `flare_t` | struct | Per-flare persistent state: screen position, fade timing, visibility, color, scene/portal identity |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `r_flareStructs` | `flare_t[128]` | global | Fixed-size pool of all flare objects |
| `r_activeFlares` | `flare_t *` | global | Singly-linked list of currently tracked flares |
| `r_inactiveFlares` | `flare_t *` | global | Free-list of unused flare slots |

## Key Functions / Methods

### R_ClearFlares
- **Signature:** `void R_ClearFlares(void)`
- **Purpose:** Resets the flare system; zeroes the pool and rebuilds the free list.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Zeroes `r_flareStructs`, sets `r_activeFlares = NULL`, repopulates `r_inactiveFlares`.
- **Calls:** `Com_Memset`
- **Notes:** Called at map load / renderer init.

### RB_AddFlare
- **Signature:** `void RB_AddFlare(void *surface, int fogNum, vec3_t point, vec3_t color, vec3_t normal)`
- **Purpose:** Called at surface tessellation time; projects a world-space point into screen space and registers or updates a flare record.
- **Inputs:** `surface` — opaque key for deduplication; `fogNum`; world `point`; `color`; optional `normal` for view-angle intensity fade.
- **Outputs/Return:** None
- **Side effects:** May allocate from `r_inactiveFlares` → `r_activeFlares`; writes `windowX/Y`, `eyeZ`, `color`, `addedFrame` on the found/new flare.
- **Calls:** `R_TransformModelToClip`, `R_TransformClipToWindow`, `VectorSubtract`, `VectorNormalizeFast`, `DotProduct`, `VectorScale`, `VectorCopy`
- **Notes:** Returns early if the projected point is outside clip space or the viewport. If no inactive slot is available the flare is silently dropped.

### RB_AddDlightFlares
- **Signature:** `void RB_AddDlightFlares(void)`
- **Purpose:** Iterates all dynamic lights in the current refdef, determines which fog volume each resides in, and submits them as flares.
- **Inputs:** None (reads `backEnd.refdef.dlights`, `tr.world->fogs`)
- **Outputs/Return:** None
- **Side effects:** Calls `RB_AddFlare` for each dlight.
- **Notes:** Gated on `r_flares->integer`; currently commented-out at the `RB_RenderFlares` call site.

### RB_TestFlare
- **Signature:** `void RB_TestFlare(flare_t *f)`
- **Purpose:** Reads one depth pixel from the framebuffer, reconstructs the scene-space Z at that pixel, and updates the flare's visibility state and `drawIntensity` fade value.
- **Inputs:** `f` — flare to test.
- **Outputs/Return:** None; writes `f->visible`, `f->fadeTime`, `f->drawIntensity`.
- **Side effects:** Issues `qglReadPixels` (forces implicit GPU sync); clears `glState.finishCalled`.
- **Notes:** Visibility threshold is a 24-unit depth tolerance. Fade rate is controlled by `r_flareFade` cvar scaled over 1 second.

### RB_RenderFlare
- **Signature:** `void RB_RenderFlare(flare_t *f)`
- **Purpose:** Emits a screen-space quad into the tessellator for a single flare.
- **Inputs:** `f` — a flare with non-zero `drawIntensity`.
- **Outputs/Return:** None
- **Side effects:** Writes directly into `tess.xyz`, `tess.texCoords`, `tess.vertexColors`, `tess.indexes`; calls `RB_BeginSurface` / `RB_EndSurface`.
- **Notes:** Size scales with viewport width and `r_flareSize` cvar; also grows as `eyeZ` approaches zero (near-field objects).

### RB_RenderFlares
- **Signature:** `void RB_RenderFlares(void)`
- **Purpose:** Main per-view flare dispatch: prunes stale flares, runs depth tests, sets up orthographic projection, and renders all visible flares.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Modifies `r_activeFlares` / `r_inactiveFlares` linked lists; pushes/pops GL matrix stacks; calls `RB_TestFlare` and `RB_RenderFlare` per active flare.
- **Notes:** Disables `GL_CLIP_PLANE0` for portal views before rendering. Documented artifact: portal flares do not occlude correctly from the main view.

## Control Flow Notes
- **Init:** `R_ClearFlares` is called at map load time.
- **Per-surface (front-end):** `RB_AddFlare` is called during surface tessellation for `SF_FLARE` surfaces.
- **Per-view (back-end, post-opaque):** `RB_RenderFlares` is called once per view after opaque geometry has been drawn, relying on the populated depth buffer.
- The system maintains cross-frame state (fade timing), which is unusual for a renderer that otherwise treats each frame as stateless.

## External Dependencies
- **Includes:** `tr_local.h` (pulls in `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `backEnd`, `tr`, `glState` — renderer globals
  - `tess` (`shaderCommands_t`) — tessellator buffer
  - `r_flares`, `r_flareSize`, `r_flareFade` — cvars
  - `R_TransformModelToClip`, `R_TransformClipToWindow` — `tr_main.c`
  - `RB_BeginSurface`, `RB_EndSurface` — `tr_shade.c`
  - `qglReadPixels`, `qglOrtho`, etc. — QGL wrappers
