# code/renderer/tr_shadows.c — Enhanced Analysis

## Architectural Role
This file implements the shadow rendering subsystem, integrated deeply into the renderer's tessellation and back-end command pipeline. It supports two techniques: **stencil shadow volumes** (per-entity silhouette-based volumes accumulated into the stencil buffer) and **projection shadows** (flat vertex deformation for lightweight blob shadows). Both are called from the geometry deformation and tessellation end-of-frame paths, making this a critical link between surface tessellation and the final stencil-masked darkening pass.

## Key Cross-References

### Incoming (who depends on this file)
- **Tessellator end path** (`tr_backend.c` / `tr_scene.c`): calls `RB_ShadowTessEnd()` when rendering surfaces marked with `SS_STENCIL_SHADOW` sort key
- **Frame finalization** (`tr_backend.c`): calls `RB_ShadowFinish()` once per frame after all opaque geometry is submitted, to apply accumulated stencil mask
- **Deformation pipeline** (`tr_backend.c` / `RB_DeformTessGeometry`): calls `RB_ProjectionShadowDeform()` when a shader stage specifies `DEFORM_PROJECTION_SHADOW`
- **Cvar system** (`qcommon/cvar.c`): reads `r_shadows` integer cvar to select shadow mode (0=off, 2=stencil shadows)

### Outgoing (what this file depends on)
- **Tessellation buffer** (`tr_local.h` `shaderCommands_t tess`): reads/writes vertex positions (`tess.xyz`), indices, vertex/index counts; extends shadow vertices into the second half of the vertex buffer
- **Back-end state** (`tr_local.h` `backEndState_t backEnd`): reads entity light direction, view parameters (mirror flag), and entity transform (axis/origin)
- **Hardware caps** (`qcommon/qcommon.h` `glConfig_t glConfig`): checks `stencilBits` to gate shadow volume rendering
- **Math utilities** (`q_math.c`): `VectorCopy`, `VectorMA`, `VectorSubtract`, `DotProduct`, `CrossProduct`
- **Renderer utilities** (`tr_*.c`): `GL_Bind` (bind white texture), `GL_State` (set blend/depth/cull state)
- **OpenGL wrappers** (`qgl.h`): all `qgl*` immediate-mode calls (`qglBegin`, `qglVertex3fv`, `qglColor3f`, `qglStencilOp`, `qglCullFace`, etc.)
- **Memory** (`qcommon/common.c`): `Com_Memset` to clear edge tables

## Design Patterns & Rationale

**Stencil Shadow Volumes** (RB_ShadowTessEnd):
- Uses the classic GPU shadow volume algorithm: extrude back-facing geometry along light direction, increment/decrement stencil on entry/exit
- **Silhouette detection via edge adjacency**: builds per-vertex directed edge lists (`edgeDefs`), then identifies silhouette edges (front-facing edges whose reverse counterpart is not front-facing)
- **Double-pass stencil** with culling reversal: first pass (back-face culling, increment) and second pass (front-face culling, decrement) to handle both sides correctly
- **Mirror special case**: reverses cull face order to maintain correct winding in mirror views — reflects architectural constraint that mirrors require special handling throughout the renderer

**Projection Shadows** (RB_ProjectionShadowDeform):
- Lightweight flat shadow: deforms vertex positions in-place to project onto a shadow plane along light direction
- **Clamping logic** (line 310-313): prevents negative shadows and excessive length by ensuring light direction has sufficient downward component
- Used for fast, simple blob shadows on characters; stored as a shader deform stage

**Silent Overflow Handling** (R_AddEdgeDef):
- Edge list silently drops overflow when `MAX_EDGE_DEFS` exceeded — defensive but non-fatal, implying this edge array size was empirically chosen and overflows are rare

## Data Flow Through This File

1. **Per-frame accumulation**:
   - `RB_ShadowTessEnd()` builds shadow volume for *one entity's tessellation*
   - Extrudes shadow vertices into upper half of `tess.xyz` buffer (assuming vertex buffer has 2× capacity)
   - Identifies front-facing triangles via light-direction dot product
   - Registers directed edges and classifies by facing
   - Issues two stencil passes (increment back-faces, decrement front-faces) to the stencil buffer
   - State left for next entity: `tess` buffer extended, stencil buffer marked

2. **End-of-frame darkening**:
   - `RB_ShadowFinish()` reads accumulated stencil (all entities), issues a full-screen darkening quad
   - Multiplies color by 0.6 wherever stencil is nonzero, darkening all shadowed pixels uniformly

3. **Deformation path** (separate from shadow volumes):
   - Shader stage triggers `RB_ProjectionShadowDeform()`
   - Reads entity shadow plane and light direction
   - Modifies `tess.xyz` in-place to project vertices onto ground plane
   - No stencil interaction; used for character blob shadows

## Learning Notes

- **Era-appropriate technique**: Stencil shadow volumes were the dominant real-time shadow method c. 1999–2010 before shadow maps/cascaded shadow maps became standard. This code represents state-of-the-art from Quake III's era.
- **CPU-intensive silhouette extraction**: Modern engines use GPU-driven shadow map rendering; this file's silhouette detection (`R_RenderShadowEdges`) is CPU-bound, requiring per-vertex edge list traversal. The disabled `#if 0` code shows an earlier brute-force path.
- **Immediate-mode OpenGL**: Uses `qglBegin`/`qglVertex3fv`/`qglEnd` throughout; would be deprecated in modern OpenGL 3.2+ and replaced with VAO/VBO rendering.
- **Defensive stencil checks**: The pattern of checking `glConfig.stencilBits < 4` appears throughout, reflecting that older hardware had no stencil buffer or insufficient precision — the code gracefully degrades.
- **Mirror handling is special**: The explicit `backEnd.viewParms.isMirror` check and culling reversal shows mirrors were a first-class concern, requiring special handling throughout the renderer.
- **Entity-local lighting model**: Shadows are per-entity (each entity has a `lightDir`); this is fundamentally different from directional/cascaded world lighting used in modern engines, limiting scalability to multiple light sources.

## Potential Issues

- **Silent edge overflow** (line 55–57): Edges beyond `MAX_EDGE_DEFS` (32) are silently dropped. No warning, assertion, or error recovery. If an entity with dense silhouettes overflows, shadows silently degrade without indication.
- **Hardcoded shadow extrusion** (line 177): The `-512` magic number for light-direction shadow extrusion is not scaled by entity size or distance, potentially causing z-fighting or incorrect shadows on very large/small entities.
- **Fixed-position darkening quad** (line 245–250): The quad is drawn at `z = -10` with identity modelview matrix; this is fragile if the near plane changes or if there are depth-buffer state issues. No depth test is disabled, which could cause incorrect interaction with opaque geometry.
- **No cvar bounds checking** (line 242): `r_shadows->integer` is read and compared against `2`, but if someone sets it to an out-of-range value, the behavior is undefined (neither condition matches, so nothing happens — but this is implicit and not documented).
- **Vertex buffer overflow assumption** (line 167): `RB_ShadowTessEnd()` checks `tess.numVertexes >= SHADER_MAX_VERTEXES / 2`, assuming the buffer is pre-allocated for 2× vertices. If this assumption is violated elsewhere in the tessellator, crashes or corruption occur.
