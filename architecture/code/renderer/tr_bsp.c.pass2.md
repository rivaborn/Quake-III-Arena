# code/renderer/tr_bsp.c — Enhanced Analysis

## Architectural Role

`tr_bsp.c` is the **renderer's sole map initialization pipeline**, sitting at the intersection of four subsystems: it consumes a raw BSP file produced by the `qcommon` filesystem, shares visibility data with the `qcommon` collision model (`CM_Load`), feeds GPU textures through the image subsystem (`tr_image.c`), and populates `tr.world` — the single global that gates all world rendering in `tr_world.c`. Nothing in this file executes per-frame; it is a pure load-time transformer that converts disk-format lumps into the renderer's internal representation and permanently commits them to the hunk.

The file also owns `R_GetEntityToken`, the bridge through which the cgame VM can sequentially walk the worldspawn entity string after the map is loaded — a rare case where the renderer serves the game logic layer rather than the other way around.

## Key Cross-References

### Incoming (who depends on this file)

| Caller | What It Uses | Source |
|--------|-------------|--------|
| `cl_cgame.c` (client layer) | `RE_LoadWorldMap` | Called when client receives a map-load configstring from the server; `RE_LoadWorldMap` is part of the `refexport_t` vtable returned by `GetRefAPI` |
| `qcommon/cm_load.c` | `RE_SetWorldVisData` | CM calls this after loading the BSP collision model to hand off its already-parsed vis buffer, avoiding a duplicate 1.8 MB allocation |
| cgame VM (`cg_main.c`) | `R_GetEntityToken` (via `trap_R_GetEntityToken`) | cgame walks the entity string post-load to locate ambient sound emitters, miscellaneous entities, and other client-side spawn points |
| `tr_world.c` | `tr.world` (the `world_t *`) | All BSP traversal, PVS culling, and surface visibility marking in `tr_world.c` is gated on `tr.world != NULL`, which is set at the end of `RE_LoadWorldMap` |
| `tr_shader.c` | `tr.lightmaps[]` | Shader stage parsing references lightmap textures by index; these are created in `R_LoadLightmaps` via `R_CreateImage` |
| `tr_light.c` / `tr_main.c` | `s_worldData.lightGridData`, `lightGridSize`, `lightGridOrigin` | Per-entity ambient/directional lighting is trilinearly sampled from the grid populated in `R_LoadLightGrid` |

### Outgoing (what this file depends on)

| Callee | Subsystem | Purpose |
|--------|-----------|---------|
| `R_SubdividePatchToGrid`, `R_GridInsertColumn/Row`, `R_FreeSurfaceGridMesh` | `tr_curve.c` | Bezier patch tessellation and LOD crack stitching |
| `R_FindShader`, `R_RemapShader` | `tr_shader.c` | Resolve disk shader names/indices to cached `shader_t` pointers |
| `R_CreateImage`, `R_SyncRenderThread` | `tr_image.c` / `tr_init.c` | Upload lightmap textures to GPU |
| `R_AllocModel` | `tr_model.c` | Allocate a `model_t` slot for each submodel (inline brush model) |
| `ri.Hunk_Alloc`, `ri.FS_ReadFile/FreeFile` | `qcommon` (via `refimport_t`) | All permanent allocations and file I/O tunnel through the `ri` import table, keeping the renderer DLL isolated from the engine's memory system |
| `ri.CM_ClusterPVS` | `qcommon/cm_test.c` | PVS queries during rendering are delegated back to the collision model |
| `COM_ParseExt`, `LittleLong`, `LittleFloat` | `q_shared.c` / `qcommon` | Entity string tokenization and endian swapping |
| `r_vertexLight`, `r_mapOverBrightBits`, `r_lightmap`, `r_fullbright`, `r_singleShader` | cvars (registered in `tr_init.c`) | Runtime quality switches consulted throughout loading |

## Design Patterns & Rationale

**Discriminated union for surface types.** Each `msurface_t` carries a `surfaceType_t` tag (`SF_FACE`, `SF_GRID`, `SF_TRIANGLES`, `SF_FLARE`, `SF_SKIP`) as the first word of its `data` pointer. `tr_surface.c` switches on this tag each frame. This pattern avoids vtables (not idiomatic C at this era) while enabling the back-end to handle heterogeneous surface types through a single dispatch.

**Shared vis buffer (`RE_SetWorldVisData`).** Rather than loading the BSP file twice, `cm_load.c` parses vis first and hands the pointer to the renderer. This is an explicit cross-subsystem memory contract: the renderer *knows* CM owns the allocation and must not free it. Modern engines solve this with reference counting; here it relies on the ordering guarantee that CM loads before the renderer.

**All-at-once hunk commitment.** The entire world is allocated from the low hunk in dependency order (shaders → lightmaps → planes → surfaces → nodes → submodels → vis). The hunk is never partially freed for world data; level reload goes through a full `RE_LoadWorldMap` which clears `s_worldData` and rebuilds from scratch. This makes the allocator trivially fast at the cost of needing a full reload on any change.

**Load-time LOD preprocessing.** Patch stitching (`R_StitchAllPatches`) and LOD error propagation (`R_FixSharedVertexLodError`) run entirely at load time. The multi-pass stitching loop (retry until no changes) is effectively a worklist algorithm — simpler to reason about than a dependency-ordered single pass, at the cost of potentially O(n²) iterations for densely connected patch groups.

## Data Flow Through This File

```
BSP file on disk (FS_ReadFile)
        │
        ▼  fileBase + lump offsets
 Raw on-disk lumps (big-endian safe via LittleLong/LittleFloat)
        │
        ├──► R_LoadShaders      → s_worldData.shaders[]   (dshader_t array, name lookups)
        ├──► R_LoadLightmaps    → tr.lightmaps[]           (GPU textures via R_CreateImage)
        ├──► R_LoadPlanes       → s_worldData.planes[]     (cplane_t, sign bits set)
        ├──► R_LoadFogs         → s_worldData.fogs[]       (fog_t, shader + bounds)
        ├──► R_LoadSurfaces     → s_worldData.surfaces[]   (msurface_t + typed data: srfSurfaceFace_t / srfGridMesh_t / srfTriangles_t / srfFlare_t)
        │       └──► R_StitchAllPatches → patch vert insertion (iterative)
        │       └──► R_FixSharedVertexLodError → LOD error propagation
        │       └──► R_MovePatchSurfacesToHunk → permanent hunk commit for grids
        ├──► R_LoadNodesAndLeafs → s_worldData.nodes[] (mnode_t BSP tree, parent-linked)
        ├──► R_LoadSubmodels    → tr.models[] (bmodel_t surface ranges)
        ├──► R_LoadVisibility   → s_worldData.vis (shared from CM or own copy)
        ├──► R_LoadEntities     → s_worldData.entityParsePoint (raw string for R_GetEntityToken)
        │       └──► gridsize/remapshader/sunvec parsing → w->lightGridSize, tr.sunDirection
        └──► R_LoadLightGrid    → s_worldData.lightGridData[] (overbright-shifted RGBA pairs)
                        │
                        ▼
              tr.world = &s_worldData   (makes world visible to tr_world.c)
```

Color shifting (`R_ColorShiftLightingBytes`) is applied at three sites: lightmap texture upload, per-vertex color on all surface types, and light grid data — ensuring consistent overbright handling across all lighting paths.

## Learning Notes

**The overbright problem.** Q3 predates HDR pipelines. The `r_mapOverBrightBits` / `tr.overbrightBits` shift system is an early attempt at HDR-like range expansion: lightmaps are stored at a normalized range but shifted left (multiplied by 2^n) before GPU upload, then compensated by the GL texture environment. This file is where that shift is applied to source data; the `R_ColorShiftLightingBytes` normalization (scale to max-component rather than saturate) preserves hue under overexposure.

**Lightmap atlas before its time.** Each 128×128 lightmap is its own texture — no atlas. Modern engines pack thousands of lightmaps into a few large textures to minimize texture-unit switches. The `tr.numLightmaps` array and per-surface `lightmapNum` index are the precursor to this concept.

**Bezier patches as first-class surfaces.** Pre-tessellating patches at load time (rather than per-frame) was an important Q3 innovation for performance. The LOD system (`widthLodError`, `heightLodError`, `lodOrigin`) allows coarser tessellation at distance with crack prevention — an early example of GPU-agnostic LOD that modern engines handle with hardware tessellation shaders.

**Inline brush models (submodels).** Each entity with `model "*N"` (doors, platforms) becomes a `bmodel_t` with a surface range in `s_worldData.surfaces`. The renderer treats these identically to world surfaces but with a separate entity transform. This is structurally equivalent to a scene-graph child node, without the overhead of a full scene graph.

**`R_GetEntityToken` as a streaming API.** The renderer maintains parse state in `s_worldData.entityParsePoint` across calls, providing a coroutine-like interface over a flat string. This is idiomatic to the Q3 era; modern engines would use a structured data format (JSON/XML) and random-access APIs.

## Potential Issues

- **Single-lightmap HACK** (`tr.numLightmaps++` at line ~145): Maps with exactly one lightmap upload a phantom second texture to work around a fullbright bug. The root cause is not identified in the code; a renderer state leak is probable.
- **`c_subdivisions` / `c_gridVerts` are non-static globals** shared with `tr_curve.c` for stat accumulation, but they are never reset between map loads, so they accumulate across multiple `RE_LoadWorldMap` calls in a session (e.g., map changes).
- **Patch stitching worst-case complexity**: The retry loop in `R_StitchAllPatches` re-iterates all `numGridMeshes` surfaces after every single column/row insertion. For densely-tiled patch terrain (common in some Q3 maps), this is O(n² × insertions). The code appears robust but load time degrades noticeably on large patch-heavy maps.
- **Entity string kept in hunk permanently**: `s_worldData.entityParsePoint` is a pointer into the hunk-allocated entity lump, which is correct — but `R_GetEntityToken` resets the pointer on exhaustion, meaning cgame callers that don't fully drain the string will discard unconsumed tokens silently.
