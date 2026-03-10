# code/renderer/tr_bsp.c

## File Purpose
Loads a Quake III BSP map file from disk and converts all its lumps into renderer-ready in-memory structures. It is the single entry point for world map loading (`RE_LoadWorldMap`) and handles all surface types, lightmaps, visibility data, fog volumes, BSP nodes/leaves, and the volumetric light grid.

## Core Responsibilities
- Parse and byte-swap all BSP lumps into the `s_worldData` (`world_t`) structure
- Upload lightmap textures to GPU with overbright color shifting
- Convert on-disk surfaces (`dsurface_t`) to typed render surfaces: planar faces, patch meshes, triangle soups, and flares
- Pre-tessellate Bezier patch meshes and stitch/fix LOD cracks between adjacent patches
- Build the BSP node/leaf tree with parent links for PVS traversal
- Load fog volumes, planes, shader references, visibility clusters, and the ambient light grid
- Allocate all world geometry into the engine hunk

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `world_t` | struct (defined in tr_local.h) | Master in-memory BSP world: nodes, surfaces, shaders, vis, light grid |
| `msurface_t` | struct | Single renderable surface with shader and typed data pointer |
| `mnode_t` | struct | BSP tree node/leaf unified; `contents == -1` means node |
| `srfSurfaceFace_t` | struct | Planar polygon face surface |
| `srfGridMesh_t` | struct | Pre-tessellated patch mesh with LOD error arrays |
| `srfTriangles_t` | struct | Arbitrary triangle-soup surface (misc_models) |
| `srfFlare_t` | struct | Lens-flare point light surface |
| `fog_t` | struct | Fog volume with bounding box and shader fog parameters |
| `bmodel_t` | struct | Submodel (inline brush model) bounding box and surface range |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_worldData` | `world_t` | static (file) | The single loaded world; pointed to by `tr.world` on success |
| `fileBase` | `byte *` | static (file) | Base pointer into the raw BSP buffer for lump offset arithmetic |
| `c_subdivisions` | `int` | global | Counter for patch subdivision stats |
| `c_gridVerts` | `int` | global | Counter for grid vertex stats |

## Key Functions / Methods

### RE_LoadWorldMap
- **Signature:** `void RE_LoadWorldMap( const char *name )`
- **Purpose:** Single public entry point; reads the BSP file and drives all sub-loaders in dependency order.
- **Inputs:** `name` — filesystem path to the `.bsp` file.
- **Outputs/Return:** void; sets `tr.world = &s_worldData` on success.
- **Side effects:** Allocates large hunk block; calls `R_SyncRenderThread`; sets `tr.worldMapLoaded`, `tr.sunDirection`; frees the raw file buffer on exit.
- **Calls:** `ri.FS_ReadFile`, `R_LoadShaders`, `R_LoadLightmaps`, `R_LoadPlanes`, `R_LoadFogs`, `R_LoadSurfaces`, `R_LoadMarksurfaces`, `R_LoadNodesAndLeafs`, `R_LoadSubmodels`, `R_LoadVisibility`, `R_LoadEntities`, `R_LoadLightGrid`, `ri.FS_FreeFile`.
- **Notes:** Guards against double-load with `tr.worldMapLoaded`. Clears `tr.world` before loading so a failed load leaves a NULL pointer rather than partial data.

### R_LoadLightmaps
- **Signature:** `static void R_LoadLightmaps( lump_t *l )`
- **Purpose:** Expands 24-bit RGB lightmap texels to 32-bit RGBA, applies overbright shift, and uploads each 128×128 tile as a GL texture.
- **Inputs:** Lump pointer into BSP buffer.
- **Side effects:** Calls `R_SyncRenderThread`; fills `tr.lightmaps[]`; uploads GPU textures via `R_CreateImage`.
- **Notes:** Skipped entirely in vertex-light or Permedia2 mode. `r_lightmap 2` replaces data with HSV intensity visualization.

### R_LoadSurfaces
- **Signature:** `static void R_LoadSurfaces( lump_t *surfs, lump_t *verts, lump_t *indexLump )`
- **Purpose:** Iterates all `dsurface_t` records and dispatches to `ParseFace`, `ParseMesh`, `ParseTriSurf`, or `ParseFlare`; runs patch stitching and LOD fixup.
- **Side effects:** Hunk-allocates all surface structs; conditionally calls `R_StitchAllPatches`, `R_FixSharedVertexLodError`, `R_MovePatchSurfacesToHunk`.

### ParseFace / ParseMesh / ParseTriSurf / ParseFlare
- **Purpose:** Convert a single `dsurface_t` to the corresponding typed surface struct; copy and byte-swap vertex data, apply color shift.
- **Notes:** `ParseMesh` calls `R_SubdividePatchToGrid` for Bezier tessellation. `ParseTriSurf` validates index range. All set `surf->data` to the allocated typed surface.

### R_StitchAllPatches / R_TryStitchingPatch / R_StitchPatches
- **Purpose:** Iteratively insert missing vertices into adjacent patch grids to eliminate highest-LOD T-junction cracks.
- **Notes:** `R_StitchPatches` returns `qtrue` after one insertion and must be called in a loop. Only operates on `SF_GRID` surfaces sharing the same `lodOrigin`/`lodRadius`.

### R_FixSharedVertexLodError / R_FixSharedVertexLodError_r
- **Purpose:** Propagate `widthLodError`/`heightLodError` values between coplanar patch grids in the same LOD group to prevent popping cracks at non-highest LOD.
- **Notes:** Recursive; marks grids with `lodFixed = 2` to avoid re-processing.

### R_LoadEntities
- **Signature:** `void R_LoadEntities( lump_t *l )`
- **Purpose:** Parses the worldspawn entity for `gridsize`, `remapshader`, and `vertexremapshader` keys; stores the raw entity string for cgame access.
- **Side effects:** Sets `w->lightGridSize`; may call `R_RemapShader`.

### R_LoadLightGrid
- **Signature:** `void R_LoadLightGrid( lump_t *l )`
- **Purpose:** Reads the ambient/directed light grid and applies overbright shift to every grid point's color pairs.
- **Side effects:** Hunk-allocates `w->lightGridData`; sets grid origin, bounds, and inverse-size vectors.

### R_GetEntityToken
- **Signature:** `qboolean R_GetEntityToken( char *buffer, int size )`
- **Purpose:** Sequential token parser over the stored entity string; rewinds on exhaustion.
- **Notes:** Called by cgame to enumerate map entities post-load.

### RE_SetWorldVisData
- **Signature:** `void RE_SetWorldVisData( const byte *vis )`
- **Purpose:** Allows the collision model (`CM_Load`) to share its already-loaded vis data instead of duplicating it.

## Control Flow Notes
- Called once during map load from the client/cgame layer (`RE_LoadWorldMap`).
- No per-frame involvement; purely an **init-phase** loader.
- `tr.world` being non-NULL is the gate for all world rendering in `tr_world.c`.
- Patch stitching and LOD fixup run entirely at load time before the hunk is committed to permanent storage via `R_MovePatchSurfacesToHunk`.

## External Dependencies
- **Includes:** `tr_local.h` → `q_shared.h`, `qfiles.h`, `qcommon.h`, `tr_public.h`, `qgl.h`
- **Defined elsewhere:**
  - `tr`, `ri`, `glConfig` — renderer globals
  - `R_SubdividePatchToGrid`, `R_GridInsertColumn`, `R_GridInsertRow`, `R_FreeSurfaceGridMesh` — `tr_curve.c`
  - `R_FindShader`, `R_RemapShader` — `tr_shader.c`
  - `R_CreateImage`, `R_SyncRenderThread` — `tr_image.c` / `tr_init.c`
  - `R_AllocModel` — `tr_model.c`
  - `COM_ParseExt`, `COM_Parse`, `LittleLong`, `LittleFloat` — `qcommon`
  - `r_vertexLight`, `r_lightmap`, `r_mapOverBrightBits`, `r_fullbright`, `r_singleShader` — cvars registered in `tr_init.c`
