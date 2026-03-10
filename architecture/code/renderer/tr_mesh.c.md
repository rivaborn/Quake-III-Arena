# code/renderer/tr_mesh.c

## File Purpose
Handles front-end rendering of MD3 triangle mesh models, including culling, LOD selection, fog membership, and submission of draw surfaces to the renderer's sort queue.

## Core Responsibilities
- Cull MD3 models against the view frustum using bounding spheres and boxes
- Compute the appropriate LOD level based on projected screen-space radius
- Determine which fog volume (if any) the model occupies
- Resolve the correct shader per surface (custom shader, skin, or embedded MD3 shader)
- Submit shadow draw surfaces (stencil and projection) for opaque surfaces
- Submit main draw surfaces to `R_AddDrawSurf` for deferred sorting and rendering
- Skip "personal model" (RF_THIRD_PERSON) surfaces unless rendering through a portal

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `md3Header_t` | struct (external) | Top-level MD3 file header; contains frame/surface counts and offsets |
| `md3Frame_t` | struct (external) | Per-frame bounding data: `localOrigin`, `radius`, `bounds[2]` |
| `md3Surface_t` | struct (external) | Per-surface geometry data including shader list and name |
| `md3Shader_t` | struct (external) | Per-surface shader reference (index into `tr.shaders`) |
| `trRefEntity_t` | struct (tr_local.h) | Entity reference including `refEntity_t e` (frame, renderfx, origin) and lighting data |
| `skin_t` | struct (tr_local.h) | Named skin overriding per-surface shaders |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `tr` | `trGlobals_t` | global | Renderer globals: current model, view params, shaders, skins, perf counters |
| `r_lodscale` | `cvar_t *` | global | Scale factor for LOD distance computation |
| `r_lodbias` | `cvar_t *` | global | Integer bias applied after LOD computation |
| `r_shadows` | `cvar_t *` | global | Shadow mode: 0=none, 2=stencil, 3=projection |

## Key Functions / Methods

### ProjectRadius
- **Signature:** `static float ProjectRadius( float r, vec3_t location )`
- **Purpose:** Projects a world-space bounding sphere radius into normalized screen space to estimate screen coverage.
- **Inputs:** `r` — world-space radius; `location` — world-space center.
- **Outputs/Return:** Normalized projected radius in [0, 1]; 0 if behind view plane.
- **Side effects:** None.
- **Calls:** `DotProduct` (macro), reads `tr.viewParms.or`, `tr.viewParms.projectionMatrix`.
- **Notes:** Manually applies the projection matrix column-major; clamps result to 1.0.

### R_CullModel
- **Signature:** `static int R_CullModel( md3Header_t *header, trRefEntity_t *ent )`
- **Purpose:** Tests the model's bounding geometry (sphere and AABB spanning current and old frame) against the view frustum.
- **Inputs:** `header` — MD3 data for the selected LOD; `ent` — entity with frame indices and axis info.
- **Outputs/Return:** `CULL_IN`, `CULL_CLIP`, or `CULL_OUT`.
- **Side effects:** Increments `tr.pc` sphere/box cull counters.
- **Calls:** `R_CullLocalPointAndRadius`, `R_CullLocalBox`.
- **Notes:** Sphere cull is skipped for entities with non-normalized axes (`nonNormalizedAxes`). When frames differ, both frame spheres are tested and only culled if both agree.

### R_ComputeLOD
- **Signature:** `int R_ComputeLOD( trRefEntity_t *ent )`
- **Purpose:** Selects an LOD index (0 = highest detail) for `tr.currentModel` based on projected screen area.
- **Inputs:** `ent` — entity with origin and frame index.
- **Outputs/Return:** Clamped LOD index in `[0, numLods-1]`.
- **Side effects:** Reads `r_lodscale->value`, `r_lodbias->integer`.
- **Calls:** `ProjectRadius`, `RadiusFromBounds`, `myftol`.
- **Notes:** If only one LOD exists, computation is entirely skipped. Objects intersecting the near plane (`ProjectRadius == 0`) default to LOD 0.

### R_ComputeFogNum
- **Signature:** `int R_ComputeFogNum( md3Header_t *header, trRefEntity_t *ent )`
- **Purpose:** Returns the index of the first world fog volume whose bounds overlap the model's bounding sphere.
- **Inputs:** `header` — MD3 header for frame offset; `ent` — entity position.
- **Outputs/Return:** Fog index (1-based), or 0 if not in any fog or `RDF_NOWORLDMODEL` is set.
- **Side effects:** None.
- **Calls:** `VectorAdd`.
- **Notes:** Fog index 0 is skipped (reserved). Comments note a FIXME for non-normalized axis handling.

### R_AddMD3Surfaces
- **Signature:** `void R_AddMD3Surfaces( trRefEntity_t *ent )`
- **Purpose:** Main entry point — validates frames, selects LOD and shader, culls, sets up lighting, and enqueues all surfaces of an MD3 model for rendering.
- **Inputs:** `ent` — the entity to render, with `tr.currentModel` already set by the caller.
- **Outputs/Return:** void.
- **Side effects:** Modifies `ent->e.frame`/`oldframe` on out-of-range values; calls `R_SetupEntityLighting`; calls `R_AddDrawSurf` (potentially multiple times per surface for shadow passes).
- **Calls:** `R_ComputeLOD`, `R_CullModel`, `R_SetupEntityLighting`, `R_ComputeFogNum`, `R_GetShaderByHandle`, `R_GetSkinByHandle`, `R_AddDrawSurf`, `ri.Printf`.
- **Notes:** Shadow surfaces are only submitted for `SS_OPAQUE` shaders with no fog and no `RF_NOSHADOW`/`RF_DEPTHHACK`. Personal models (RF_THIRD_PERSON outside portals) skip the main draw surface submission but shadows with mode > 1 still invoke lighting setup.

## Control Flow Notes
Called from the renderer front-end scene traversal (likely `R_AddEntitySurfaces` in `tr_scene.c` or `tr_main.c`) once per MD3 entity per frame. Runs entirely on the front-end thread before the sort and back-end draw pass. No rendering commands are issued here — only `drawSurf_t` entries are enqueued via `R_AddDrawSurf`.

## External Dependencies
- **Includes:** `tr_local.h` (transitively includes `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`)
- **Defined elsewhere:**
  - `R_CullLocalPointAndRadius`, `R_CullLocalBox` — `tr_main.c`
  - `R_AddDrawSurf` — `tr_main.c`
  - `R_SetupEntityLighting` — `tr_light.c`
  - `R_GetShaderByHandle`, `R_GetSkinByHandle` — `tr_shader.c` / `tr_image.c`
  - `RadiusFromBounds` — `q_shared.c` / math library
  - `myftol` — platform-specific (x86 asm or cast macro)
  - `tr`, `r_lodscale`, `r_lodbias`, `r_shadows` — `tr_init.c` / `tr_main.c`
