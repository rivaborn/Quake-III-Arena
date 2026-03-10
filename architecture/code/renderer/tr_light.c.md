# code/renderer/tr_light.c

## File Purpose
Handles dynamic and static lighting calculations for the Quake III Arena renderer. It computes per-entity lighting by sampling the world light grid (trilinear interpolation) and accumulating dynamic light contributions, then stores results used by the shader backend.

## Core Responsibilities
- Transform dynamic light (dlight) origins into local entity space
- Determine which dlights intersect a bmodel's bounding box and mark affected surfaces
- Sample the world light grid via trilinear interpolation to compute ambient and directed light for entities
- Accumulate dlight contributions into per-entity lighting vectors
- Expose a public API for querying lighting at an arbitrary world point

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `dlight_t` | struct (defined in tr_local.h) | Dynamic light: world origin, color, radius, transformed local origin |
| `trRefEntity_t` | struct (defined in tr_local.h) | Renderer-side entity carrying computed `ambientLight`, `directedLight`, `lightDir`, `ambientLightInt` |
| `bmodel_t` | struct (defined in tr_local.h) | BSP inline model with bounds and surface list used for dlight intersection |
| `orientationr_t` | struct (defined in tr_local.h) | Entity orientation (origin + axis matrix) used for coordinate transforms |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `r_ambientScale` | `cvar_t *` | global (extern) | Scale factor applied to grid-sampled ambient light |
| `r_directedScale` | `cvar_t *` | global (extern) | Scale factor applied to grid-sampled directed light |
| `r_debugLight` | `cvar_t *` | global (extern) | When non-zero, prints ambient/directed max values for first-person entity |

## Key Functions / Methods

### R_TransformDlights
- **Signature:** `void R_TransformDlights( int count, dlight_t *dl, orientationr_t *or )`
- **Purpose:** Projects each dlight's world-space origin into the local coordinate system described by `or`, storing the result in `dl->transformed`.
- **Inputs:** `count` dlights starting at `dl`; orientation `or` (origin + 3-axis matrix)
- **Outputs/Return:** void; writes `dl->transformed` for each dlight in-place
- **Side effects:** Modifies `dlight_t::transformed` array elements
- **Calls:** `VectorSubtract`, `DotProduct`
- **Notes:** Called by both `R_DlightBmodel` (front end) and the back end before lighting calculations.

### R_DlightBmodel
- **Signature:** `void R_DlightBmodel( bmodel_t *bmodel )`
- **Purpose:** Determines which active dlights overlap a bmodel's AABB and stamps a bitmask into the `dlightBits` field of every SF_FACE, SF_GRID, and SF_TRIANGLES surface it contains.
- **Inputs:** `bmodel` — the inline BSP model to test
- **Outputs/Return:** void; sets `tr.currentEntity->needDlights` and per-surface `dlightBits[tr.smpFrame]`
- **Side effects:** Reads `tr.refdef.dlights`, `tr.or`; writes to surface structs and `tr.currentEntity->needDlights`
- **Calls:** `R_TransformDlights`
- **Notes:** Dlight index `i` maps to bit `1 << i`; mask limited to 32 dlights. Uses the SMP-safe double-buffered `smpFrame` index.

### R_SetupEntityLightingGrid
- **Signature:** `static void R_SetupEntityLightingGrid( trRefEntity_t *ent )`
- **Purpose:** Trilinearly interpolates the world light grid at the entity's position to produce `ambientLight`, `directedLight`, and `lightDir` vectors.
- **Inputs:** `ent` — entity whose `e.origin` or `e.lightingOrigin` is used
- **Outputs/Return:** void; writes `ent->ambientLight`, `ent->directedLight`, `ent->lightDir`
- **Side effects:** Reads `tr.world->lightGridData`, `tr.sinTable`; clamps pos to grid bounds
- **Calls:** `VectorCopy`, `VectorSubtract`, `VectorClear`, `VectorMA`, `VectorNormalize2`, `VectorScale`, `assert`
- **Notes:** Grid stores 8 bytes per cell: 3 ambient RGB, 3 directed RGB, 2 encoded normal angles (lat/lng). Samples in walls (all-zero ambient+directed) are skipped; `totalFactor` renormalises if any samples were skipped. `#if idppc` path separates float loads to avoid PPC LHS stalls.

### LogLight
- **Signature:** `static void LogLight( trRefEntity_t *ent )`
- **Purpose:** Debug helper that prints the max component of ambient and directed light for the first-person entity.
- **Inputs:** `ent`
- **Side effects:** Calls `ri.Printf`; no-op unless `RF_FIRST_PERSON` is set
- **Notes:** Trivial debug utility; only active when `r_debugLight->integer` is set.

### R_SetupEntityLighting
- **Signature:** `void R_SetupEntityLighting( const trRefdef_t *refdef, trRefEntity_t *ent )`
- **Purpose:** Full per-entity lighting setup: samples light grid (or falls back to sun/identity), adds dlight contributions to directed light, clamps ambient, packs `ambientLightInt`, and transforms `lightDir` into entity-local space.
- **Inputs:** `refdef` — current scene definition; `ent` — entity to light
- **Outputs/Return:** void; populates all lighting fields of `ent`
- **Side effects:** Sets `ent->lightingCalculated = qtrue`; reads `tr.world->lightGridData`, `tr.identityLight`, `tr.sunDirection`
- **Calls:** `R_SetupEntityLightingGrid`, `VectorLength`, `VectorScale`, `VectorSubtract`, `VectorNormalize`, `VectorMA`, `myftol`, `DotProduct`, `LogLight`
- **Notes:** Guard `ent->lightingCalculated` prevents redundant work per frame. Dlight power formula: `power = DLIGHT_AT_RADIUS * radius²`, attenuated as `power / d²` with minimum distance clamp of 16 units.

### R_LightForPoint
- **Signature:** `int R_LightForPoint( vec3_t point, vec3_t ambientLight, vec3_t directedLight, vec3_t lightDir )`
- **Purpose:** Public API to query light grid values at an arbitrary world point without a full entity.
- **Inputs:** `point` — world position to sample
- **Outputs/Return:** `qtrue` on success, `qfalse` if `lightGridData` is NULL (no-light map)
- **Side effects:** Allocates a temporary `trRefEntity_t` on the stack; calls `R_SetupEntityLightingGrid`
- **Calls:** `Com_Memset`, `VectorCopy`, `R_SetupEntityLightingGrid`
- **Notes:** Documented workaround for `-nolight` maps causing a NULL dereference (bk010103).

## Control Flow Notes
- Called during the **front-end** scene traversal: `R_DlightBmodel` is invoked from `R_AddBrushModelSurfaces`; `R_SetupEntityLighting` is called per entity before surfaces are added.
- `R_TransformDlights` is also called by the **back end** (before per-surface dlight shading) using `backEnd.or`.
- No render or frame init/shutdown logic resides here.

## External Dependencies
- `tr_local.h` — all renderer types, `tr`, `backEnd`, `ri`, math macros
- `VectorSubtract`, `DotProduct`, `VectorCopy`, `VectorClear`, `VectorMA`, `VectorScale`, `VectorNormalize`, `VectorNormalize2`, `VectorLength` — defined in shared math library
- `Com_Memset` — defined in `qcommon`
- `ri.Printf` — renderer import table, defined elsewhere
- `myftol` — fast float-to-int, platform-specific (inline or asm)
- `r_ambientScale`, `r_directedScale`, `r_debugLight` — cvars registered in `tr_init.c`
- `tr.sinTable` — precomputed in `tr_init.c`
- `FUNCTABLE_SIZE`, `FUNCTABLE_MASK` — constants from `tr_local.h`
