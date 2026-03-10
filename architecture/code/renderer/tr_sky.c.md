# code/renderer/tr_sky.c

## File Purpose
Implements sky and cloud rendering for Quake III Arena's renderer backend. It handles sky polygon clipping to a cube box, generation of subdivided sky box geometry, cloud layer vertex generation with spherical projection, and sun quad rendering.

## Core Responsibilities
- Clip world-space sky polygons onto the 6 faces of a sky cube box to determine which sky face regions need drawing
- Generate subdivided mesh vertices and texture coordinates for the sky box outer shell
- Compute cloud layer texture coordinates using a spherical intersection formula (called once at shader parse time)
- Populate `tess` (the tessellator) with cloud geometry vertices and indices per-frame
- Draw the sky box outer faces directly via immediate-mode OpenGL (`qglBegin`/`qglEnd`)
- Render the sun as a billboard quad aligned to `tr.sunDirection`
- Act as the sky shader stage iterator (`RB_StageIteratorSky`), orchestrating the full sky draw sequence

## Key Types / Data Structures
None declared in this file; uses types from `tr_local.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `shaderCommands_t` (tess) | struct (extern) | Tessellator buffer holding current surface's vertices/indexes; cloud geometry is written here |
| `shader_t` / `skyParms_t` | struct | Holds `outerbox[6]` image pointers and `cloudHeight` used during sky draw |
| `viewParms_t` | struct | Provides `zFar` (box sizing) and `or.origin` (view position) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_cloudTexCoords` | `float[6][9][9][2]` | static (file) | Precomputed cloud layer UV coordinates per sky face subdivision, built by `R_InitSkyTexCoords` |
| `s_cloudTexP` | `float[6][9][9]` | static (file) | Parametric intersection distance used during cloud coord precomputation |
| `sky_clip[6]` | `vec3_t[]` | static (file) | Clipping plane normals for the 6-plane convex sky clipper |
| `sky_mins[2][6]` / `sky_maxs[2][6]` | `float` | static (file) | Per-face min/max ST extents accumulated during polygon clipping |
| `sky_min` / `sky_max` | `float` | static (file) | UV clamping bounds to avoid bilinear seams (set to `1/256` and `255/256` for clouds, `0`/`1` for box) |
| `sky_texorder[6]` | `int[]` | static (file) | Maps sky face index to `outerbox[]` texture slot order |
| `s_skyPoints` / `s_skyTexCoords` | `vec3_t[9][9]` / `float[9][9][2]` | static (file) | Per-frame scratch buffers for sky box and cloud face vertex positions and UVs |

## Key Functions / Methods

### AddSkyPolygon
- **Signature:** `static void AddSkyPolygon(int nump, vec3_t vecs)`
- **Purpose:** Projects a clipped polygon's vertices onto a cube face and expands that face's `sky_mins`/`sky_maxs` ST bounds.
- **Inputs:** Vertex count and flat array of 3D positions (view-relative).
- **Outputs/Return:** void; updates `sky_mins`/`sky_maxs` globals.
- **Side effects:** Writes to file-static `sky_mins`, `sky_maxs`.
- **Calls:** `VectorAdd`, `fabs`.
- **Notes:** Uses `vec_to_st[6][3]` lookup to determine which cube face and how to map XYZ to ST.

### ClipSkyPolygon
- **Signature:** `static void ClipSkyPolygon(int nump, vec3_t vecs, int stage)`
- **Purpose:** Recursively clips a polygon against each of the 6 `sky_clip` planes; at stage 6 calls `AddSkyPolygon`.
- **Inputs:** Vertex count, flat vertex array, current clip stage index (0–6).
- **Outputs/Return:** void.
- **Side effects:** Recursive; ultimately writes `sky_mins`/`sky_maxs` via `AddSkyPolygon`.
- **Calls:** `DotProduct`, `VectorCopy`, `AddSkyPolygon`, itself recursively.
- **Notes:** `MAX_CLIP_VERTS = 64`; errors fatally if exceeded.

### RB_ClipSkyPolygons
- **Signature:** `void RB_ClipSkyPolygons(shaderCommands_t *input)`
- **Purpose:** Entry point that resets sky bounds and clips all triangles in the tessellator against the sky cube.
- **Inputs:** Current tessellator surface.
- **Side effects:** Resets and then populates `sky_mins`/`sky_maxs` via `ClearSkyBox` + `ClipSkyPolygon`.
- **Calls:** `ClearSkyBox`, `VectorSubtract`, `ClipSkyPolygon`.

### MakeSkyVec
- **Signature:** `static void MakeSkyVec(float s, float t, int axis, float outSt[2], vec3_t outXYZ)`
- **Purpose:** Converts a normalized ST coordinate on a cube face to a 3D world-space point and optionally a clamped UV for rendering.
- **Inputs:** `s`,`t` in [-1,1], cube face `axis`, output pointers (UV may be NULL).
- **Outputs/Return:** Fills `outXYZ`; optionally fills `outSt`.
- **Side effects:** Reads `backEnd.viewParms.zFar` and file-static `sky_min`/`sky_max`.
- **Calls:** None (inline math).
- **Notes:** Box size = `zFar / 1.75` (approximates `zFar / sqrt(3)`).

### DrawSkyBox
- **Signature:** `static void DrawSkyBox(shader_t *shader)`
- **Purpose:** Generates subdivided geometry for each visible sky face and submits it as GL triangle strips.
- **Inputs:** Sky shader (for `outerbox` images).
- **Side effects:** Issues `qglBegin`/`qglEnd` draws; writes `s_skyPoints`/`s_skyTexCoords`.
- **Calls:** `MakeSkyVec`, `DrawSkySide`, `GL_Bind`, `qglBegin`, `qglEnd`, `qglTexCoord2fv`, `qglVertex3fv`.

### R_BuildCloudData
- **Signature:** `void R_BuildCloudData(shaderCommands_t *input)`
- **Purpose:** Populates `tess` with cloud geometry for all active shader stages; later drawn by `RB_StageIteratorGeneric`.
- **Inputs:** Current tessellator (must be a sky shader).
- **Side effects:** Resets and fills `tess.numVertexes`, `tess.numIndexes`, `tess.xyz`, `tess.texCoords`.
- **Calls:** `FillCloudBox`.

### R_InitSkyTexCoords
- **Signature:** `void R_InitSkyTexCoords(float heightCloud)`
- **Purpose:** Precomputes spherical cloud-layer UV coordinates for all 6 faces × 9×9 subdivisions. Called once when a sky shader is parsed.
- **Inputs:** Cloud layer height above the virtual ground sphere (radius 4096).
- **Side effects:** Fills `s_cloudTexCoords` and `s_cloudTexP` globals.
- **Calls:** `MakeSkyVec`, `DotProduct`, `sqrt`, `VectorScale`, `VectorNormalize`, `Q_acos`.
- **Notes:** Sets `backEnd.viewParms.zFar = 1024` temporarily so `MakeSkyVec` is usable before a world is loaded.

### RB_DrawSun
- **Signature:** `void RB_DrawSun(void)`
- **Purpose:** Draws the sun as a billboard quad aligned perpendicular to `tr.sunDirection` at the far clip distance.
- **Inputs:** None (reads `backEnd`, `tr.sunDirection`, `r_drawSun` cvar).
- **Side effects:** Issues `RB_BeginSurface`/`RB_EndSurface`; changes depth range and matrix state.
- **Calls:** `qglLoadMatrixf`, `qglTranslatef`, `PerpendicularVector`, `CrossProduct`, `VectorScale`, `RB_BeginSurface`, `RB_EndSurface`, `qglDepthRange`.

### RB_StageIteratorSky
- **Signature:** `void RB_StageIteratorSky(void)`
- **Purpose:** Top-level sky render function called by the surface dispatch system. Sequences polygon clipping → outer box draw → cloud data generation → generic stage iteration.
- **Inputs:** None (operates on global `tess`).
- **Side effects:** Sets `backEnd.skyRenderedThisView = qtrue`; issues GL depth range changes; calls multiple subsystems.
- **Calls:** `RB_ClipSkyPolygons`, `DrawSkyBox`, `R_BuildCloudData`, `RB_StageIteratorGeneric`, `qglDepthRange`, `GL_State`, `qglTranslatef`, `qglPushMatrix`, `qglPopMatrix`, `qglColor3f`.

## Control Flow Notes
This file is purely a **render** stage. `RB_StageIteratorSky` is the entry point, called during the backend draw-surface pass when a surface's shader has `optimalStageIteratorFunc = RB_StageIteratorSky`. Sky box faces are drawn immediately via OpenGL immediate mode. Cloud geometry is batched into `tess` and flushed through `RB_StageIteratorGeneric`. The sun is drawn separately after the sky pass in `RB_DrawSun`, guarded by `backEnd.skyRenderedThisView`.

## External Dependencies
- **`tr_local.h`** — all renderer types, `tess`, `backEnd`, `tr`, cvars (`r_fastsky`, `r_drawSun`, `r_showsky`)
- **Defined elsewhere:** `RB_StageIteratorGeneric`, `RB_BeginSurface`, `RB_EndSurface`, `GL_Bind`, `GL_State`, `PerpendicularVector`, `CrossProduct`, `Q_acos`, `myftol`, `ri.Error`, all `qgl*` OpenGL wrappers
