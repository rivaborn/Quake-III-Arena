# code/renderer/tr_shadows.c

## File Purpose
Implements real-time shadow rendering for Quake III Arena's renderer backend. Provides two shadow techniques: stencil-buffer shadow volumes (for per-entity silhouette shadows) and flat projection shadow deformation (for planar blob shadows cast onto surfaces).

## Core Responsibilities
- Build per-vertex edge definition lists from tessellated geometry
- Determine which triangles face the light source
- Identify silhouette edges (edges shared only by front-facing triangles)
- Render shadow volume geometry into the stencil buffer (increment/decrement passes)
- Apply a full-screen darkening quad to pixels marked by the stencil buffer
- Deform vertex positions to project geometry flat onto a shadow plane for projection shadows

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `edgeDef_t` | struct | Stores one directed edge: destination vertex index (`i2`) and whether the owning triangle faces the light (`facing`) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `edgeDefs` | `edgeDef_t [SHADER_MAX_VERTEXES][MAX_EDGE_DEFS]` | static (file) | Per-vertex adjacency list of directed edges, rebuilt each shadow call |
| `numEdgeDefs` | `int [SHADER_MAX_VERTEXES]` | static (file) | Count of edges registered for each vertex |
| `facing` | `int [SHADER_MAX_INDEXES/3]` | static (file) | Per-triangle light-facing flag (1 = faces light, 0 = back-facing) |

## Key Functions / Methods

### R_AddEdgeDef
- **Signature:** `void R_AddEdgeDef( int i1, int i2, int facing )`
- **Purpose:** Appends a directed edge from vertex `i1` to `i2` with the given facing flag into `edgeDefs[i1]`.
- **Inputs:** Source vertex index `i1`, destination vertex index `i2`, triangle facing flag.
- **Outputs/Return:** None.
- **Side effects:** Mutates `edgeDefs` and `numEdgeDefs` file-static arrays.
- **Calls:** None.
- **Notes:** Silently drops edges when `MAX_EDGE_DEFS` (32) is exceeded — overflow is ignored.

### R_RenderShadowEdges
- **Signature:** `void R_RenderShadowEdges( void )`
- **Purpose:** Iterates all registered directed edges and emits a GL_TRIANGLE_STRIP quad (original vertex + extruded shadow vertex) for every silhouette edge — i.e., a front-facing edge whose reverse counterpart has no front-facing twin.
- **Inputs:** Reads `edgeDefs`, `numEdgeDefs`, `tess.xyz`, `tess.numVertexes`.
- **Outputs/Return:** None.
- **Side effects:** Issues immediate-mode OpenGL draw calls (`qglBegin`/`qglVertex3fv`/`qglEnd`).
- **Calls:** `qglBegin`, `qglVertex3fv`, `qglEnd`.
- **Notes:** Contains a disabled `#if 0` brute-force path that renders all front-facing triangle edges; the active path is the proper silhouette-only path. Counter variables `c_edges` / `c_rejected` are computed but not exported.

### RB_ShadowTessEnd
- **Signature:** `void RB_ShadowTessEnd( void )`
- **Purpose:** Main shadow volume entry point called at tessellation end. Extrudes shadow vertices along the negated light direction, classifies triangle facing, builds edge lists, then renders the shadow volume twice (back-face increment, front-face decrement) into the stencil buffer.
- **Inputs:** `tess` (current tessellation state), `backEnd.currentEntity->lightDir`, `backEnd.viewParms.isMirror`, `glConfig.stencilBits`.
- **Outputs/Return:** None.
- **Side effects:** Writes extruded positions into `tess.xyz[numVertexes..2*numVertexes-1]`; modifies stencil buffer and OpenGL state (cull face, stencil test/op, color mask).
- **Calls:** `VectorCopy`, `VectorMA`, `Com_Memset`, `VectorSubtract`, `CrossProduct`, `DotProduct`, `R_AddEdgeDef`, `GL_Bind`, `qglEnable`, `GL_State`, `qglColor3f`, `qglColorMask`, `qglStencilFunc`, `qglCullFace`, `qglStencilOp`, `R_RenderShadowEdges`.
- **Notes:** Guards against vertex buffer overflow (`>= SHADER_MAX_VERTEXES / 2`) and insufficient stencil precision (`< 4 bits`). Mirror views reverse the cull order to maintain correct winding.

### RB_ShadowFinish
- **Signature:** `void RB_ShadowFinish( void )`
- **Purpose:** After all shadow volumes for a frame are rendered, darkens all pixels with non-zero stencil by drawing a large full-screen quad with a multiply blend.
- **Inputs:** `r_shadows->integer`, `glConfig.stencilBits`.
- **Outputs/Return:** None.
- **Side effects:** Issues full-screen GL_QUADS draw with DST_COLOR blend; disables stencil test.
- **Calls:** `qglEnable`, `qglStencilFunc`, `qglDisable`, `GL_Bind`, `qglLoadIdentity`, `qglColor3f`, `GL_State`, `qglBegin`, `qglVertex3f`, `qglEnd`, `qglColor4f`.
- **Notes:** Only executes when `r_shadows == 2`. The quad is drawn at z = -10 in identity modelview space, which is a fixed-position hack for the darkening pass.

### RB_ProjectionShadowDeform
- **Signature:** `void RB_ProjectionShadowDeform( void )`
- **Purpose:** Deforms tessellation vertices in-place to project them flat onto the entity's shadow plane along the light direction, producing a planar blob shadow.
- **Inputs:** `tess.xyz`, `tess.numVertexes`, `backEnd.or.axis`, `backEnd.or.origin`, `backEnd.currentEntity->e.shadowPlane`, `backEnd.currentEntity->lightDir`.
- **Outputs/Return:** None.
- **Side effects:** Overwrites `tess.xyz` positions directly.
- **Calls:** `DotProduct`, `VectorCopy`, `VectorMA`.
- **Notes:** Clamps the light direction to prevent shadows from going negative or becoming excessively long when `d < 0.5`. Used by the `DEFORM_PROJECTION_SHADOW` shader deform path.

## Control Flow Notes
- `RB_ShadowTessEnd` is called by the tessellator end-of-surface path when the active shader is a shadow volume shader (sort `SS_STENCIL_SHADOW`).
- `RB_ShadowFinish` is called once per frame after all geometry is submitted, to apply the accumulated stencil mask.
- `RB_ProjectionShadowDeform` is invoked from `RB_DeformTessGeometry` when a shader stage specifies `DEFORM_PROJECTION_SHADOW`.

## External Dependencies
- **Includes:** `tr_local.h` (pulls in all renderer types, `tess`, `backEnd`, `tr`, `glConfig`, `r_shadows`)
- **Defined elsewhere:** `tess` (`shaderCommands_t`), `backEnd` (`backEndState_t`), `tr.whiteImage`, `glConfig.stencilBits`, `r_shadows` cvar, all `qgl*` OpenGL wrappers, math macros (`VectorCopy`, `VectorMA`, `DotProduct`, `CrossProduct`, `VectorSubtract`), `Com_Memset`, `GL_Bind`, `GL_State`
