# code/renderer/tr_shader.c — Enhanced Analysis

## Architectural Role

`tr_shader.c` is the renderer's **material registry and compiler**. It occupies the boundary between the data layer (`.shader` text files loaded from the VFS via `ri.FS_*`) and the execution layer (`tr_shade.c` iterators, `tr_backend.c` command queue). Every draw-surface sort key that the front-end produces and the back-end consumes encodes a shader index into `tr.sortedShaders[]` — all entries in that array originate here via `GeneratePermanentShader`. This file therefore gates every pixel the renderer draws.

Its secondary role is an **optimizer**: it detects common two-pass lightmap+diffuse patterns at registration time and collapses them into a single multitexture pass, a critical 1999–2001 era optimization given hardware TMU limits. It also selects the correct `RB_StageIterator*` function pointer for each shader at finalization time, dispatching work efficiently before the render loop starts.

## Key Cross-References

### Incoming (who depends on this file)

- **`tr_init.c / R_Init`** — calls `R_InitShaders` once at renderer startup; `tr.defaultShader`, `tr.shadowShader`, `tr.flareShader`, `tr.sunShader`, and `tr.projectionShadowShader` are all manufactured here and stored in `trGlobals_t tr`
- **`tr_bsp.c`** — calls `R_FindShader` for every surface face while loading a BSP map, providing the per-face lightmap index; this is the highest-volume call site
- **`tr_model.c`** — calls `RE_RegisterShader` / `RE_RegisterShaderNoMip` for MD3 surface materials and skin files
- **`tr_main.c / tr_scene.c`** — read `tr.sortedShaders[]` (populated by `SortNewShader`) and the `shader_t.sort` key to drive the draw-surface sort and flush
- **`tr_backend.c`** — accesses `drawSurf_t.sort` fields that encode the shader sorted index; `FixRenderCommandList` patches these fields in the in-flight `backEndData[tr.smpFrame]->commands` buffer during shader registration
- **Client / cgame VM** — reaches `RE_RegisterShader*` and `R_RemapShader` through the `refexport_t` vtable; cgame calls `trap_R_RegisterShader` and `trap_R_RemapShader` which route here

### Outgoing (what this file depends on)

- **`tr_shade.c`** — `ComputeStageIteratorFunc` assigns `shader.optimalStageIteratorFunc` from `{RB_StageIteratorGeneric, RB_StageIteratorSky, RB_StageIteratorVertexLitTexture, RB_StageIteratorLightmappedMultitexture}`; these are defined in `tr_shade.c` and are the sole execution path for drawing the shader
- **`tr_image.c`** — `R_FindImageFile` is called from `ParseStage` for every `map`, `clampMap`, and `animMap` directive; also called from `R_FindShader` to synthesize implicit shaders for images lacking a text definition
- **`tr_sky.c`** — `R_InitSkyTexCoords` is called from `ParseSkyParms`
- **`tr_backend.c`** — `R_SyncRenderThread` is called by `R_FindShader` before modifying shared state in SMP mode
- **`qcommon` parse layer** — `COM_ParseExt`, `SkipBracedSection`, `SkipRestOfLine`, `COM_Compress`, `COM_StripExtension`, `COM_DefaultExtension` (all in `qcommon/`)
- **`ri` import table** — `ri.FS_ListFiles`, `ri.FS_ReadFile`, `ri.FS_FreeFile`, `ri.Hunk_Alloc`, `ri.CIN_PlayCinematic`; essentially the entire VFS and memory subsystem is accessed only through this indirection
- **`tr` global** — reads and writes `trGlobals_t tr` (defined in `tr_init.c`): `tr.shaders[]`, `tr.sortedShaders[]`, `tr.numShaders`, `tr.defaultShader`, `tr.sunLight`, `tr.sunDirection`, and others
- **`glConfig`** — reads `glConfig.hardwareType` (for Voodoo3/Permedia2 multitexture workarounds) and `glConfig.textureEnvCombineAvailable`

## Design Patterns & Rationale

**Parse-to-workspace-then-promote.** The global `shader`, `stages`, and `texMods` statics serve as a scratch workspace during parsing. Only once a shader is fully validated does `GeneratePermanentShader` deep-copy it to hunk memory. This avoids incremental small allocations in the hunk allocator — a critical concern in Q3's dual-ended hunk model where allocation failure is a fatal error and fragmentation cannot be recovered.

**Two-level hash tables with different sizes.** `hashTable[1024]` indexes registered `shader_t*` instances by name (runtime lookup). `shaderTextHashTable[2048]` indexes raw text-block pointers within `s_shaderText` (parse-time lookup). The larger text table reflects that text lookup is a hot path during map load (every BSP surface triggers it), while runtime lookup by name is less frequent. `shaderTextHashTable` stores raw `char**` pointers into the contiguous `s_shaderText` buffer — no copies, which means `s_shaderText` must remain alive for the session.

**Data-driven multitexture collapse.** The `collapse[]` table maps `(srcBlend0, dstBlend0, srcBlend1, dstBlend1)` pairs to `GL_MODULATE`/`GL_ADD` env modes. This separates the recognition logic from the optimization action, making it easy to audit which two-pass combos are hardware-acceleratable. The explicit Voodoo3 comment (ensure lightmap is in `bundle[1]`) reflects real hardware constraints of the era.

**Sort key patching under SMP.** `FixRenderCommandList` is an unusual pattern: it walks the back-end command buffer at registration time to patch sort indices for in-flight draw calls. This exists because the renderer may be rendering frame N on the back-end thread while the front-end registers new shaders for frame N+1, and `SortNewShader` shifts all indices ≥ the insertion point. Modern renderers avoid this by finalizing sort keys before handing off to the GPU or by using stable identifiers rather than ordinal positions.

## Data Flow Through This File

```
ri.FS_ListFiles("scripts/*.shader")
        ↓
ri.FS_ReadFile → COM_Compress (in-place) → concatenate → s_shaderText (hunk)
        ↓
Build shaderTextHashTable[2048]: name → char* pointer into s_shaderText
        ↓
R_FindShader(name, lightmapIndex, mipFlag)
  → hashTable lookup (early exit if already registered)
  → FindShaderInShaderText → O(1) shaderTextHashTable hit
  → ParseShader (fills global `shader` / `stages`)
     → ParseStage → R_FindImageFile (uploads texture, returns image_t*)
     → ParseDeform / ParseSkyParms / ParseSurfaceParm
  → FinishShader:
     → CollapseMultitexture (may merge stages[0]+stages[1])
     → VertexLightingCollapse (if r_vertexLight)
     → ComputeStageIteratorFunc (assigns function pointer)
     → GeneratePermanentShader:
          → ri.Hunk_Alloc(shader_t)
          → ri.Hunk_Alloc(shaderStage_t* × numStages)
          → ri.Hunk_Alloc(texModInfo_t per stage)
          → SortNewShader → FixRenderCommandList (patches backEndData)
          → hashTable[hash] insert
          → tr.shaders[tr.numShaders++]
        ↓
Returns shader_t* (permanent, hunk-resident)
```

## Learning Notes

- **Forerunner of modern material graphs.** The `.shader` text format — multi-pass, per-stage blend modes, waveform generators, texture coordinate modifiers — is a direct predecessor to Unreal Engine's material editor and Unity's ShaderLab. Q3's innovation was making this fully data-driven at runtime, not baked at compile time.

- **`deferLoad` is set but never consumed** in this file (only assigned via `r_lazyShaders` cvar). This suggests an incomplete deferred/lazy shader loading feature; it's a real-world example of feature creep stopped mid-implementation.

- **Non-reentrant by design.** The global workspace statics (`shader`, `stages`, `texMods`) make `ParseShader` non-reentrant. This was acceptable in 1999 because shader registration happened during map load (single-threaded), and the SMP renderer only separates front-end scene traversal from back-end GL submission — not shader registration.

- **`tr.numShaders` is a hard ceiling.** `tr.shaders[]` is a fixed array (`MAX_SHADERS = 1024`). Mods that define many custom materials can hit this limit. Modern engines use dynamic arrays or hash-map-only storage.

- **Sort key stability assumption.** The entire draw-surface sorting system (`tr_main.c`) assumes shader sorted indices are stable once assigned. `FixRenderCommandList` is the only mechanism that maintains this invariant under concurrent registration + rendering. If a mod triggers shader registration mid-frame at high volume, this O(cmd_count) walk happens repeatedly — a subtle performance cliff.

- **Image upload happens during shader registration.** `R_FindImageFile` in `ParseStage` uploads textures to the GPU synchronously. This couples the parsing pipeline to GL state, preventing background loading without significant restructuring — something modern streaming engines solve with staging queues and resident textures.

## Potential Issues

- **`shaderTextHashTable` collision resolution is absent.** Collisions simply overwrite the slot (the hash table stores only one pointer per bucket). Shaders with colliding names will silently shadow each other during `FindShaderInShaderText`. The `hashTable` for registered shaders uses chaining (`sh->next`) and avoids this, but the text lookup table does not.
- **`FixRenderCommandList` walks all commands on every new shader.** During BSP map loading (hundreds of shaders registered sequentially), this is called hundreds of times, each time walking the command list. Under SMP this runs on the front-end thread while the back-end may be simultaneously reading the same buffer — the synchronization relies solely on the `R_SyncRenderThread` call in `R_FindShader`, but `FixRenderCommandList` itself does not re-acquire any lock before writing.
- **`deferLoad` dead code** could mislead future maintainers into believing lazy loading is active when it is not.
