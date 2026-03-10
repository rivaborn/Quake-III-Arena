# code/renderer/tr_shader.c

## File Purpose
Parses and manages all shader definitions for the Quake III Arena renderer. It handles loading `.shader` text files, parsing their syntax into `shader_t`/`shaderStage_t` structures, optimizing multi-pass shaders (multitexture collapsing, vertex lighting), and maintaining a hash-table registry of all loaded shaders.

## Core Responsibilities
- Load and concatenate all `.shader` script files from the `scripts/` directory into a single in-memory text buffer
- Parse shader text blocks into the global `shader`/`stages` workspace, then promote to permanent hunk-allocated instances
- Resolve shader lookups by name and lightmap index, creating implicit default shaders for unmapped images
- Optimize shaders: collapse two-pass modulate/add combos into single multitexture passes; apply vertex-lighting collapse when hardware demands it
- Maintain two hash tables: one for registered `shader_t*` instances, one for fast text-block lookup by name
- Provide public registration entry points (`RE_RegisterShader`, `RE_RegisterShaderLightMap`, `RE_RegisterShaderNoMip`)
- Remap shaders at runtime via `R_RemapShader`
- Fix in-flight render command lists when new shaders shift sorted indices

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `shader_t` | struct | Complete shader descriptor: stages, sort key, flags, sky/fog params, deform stages |
| `shaderStage_t` | struct | Per-pass state: texture bundles, rgbGen, alphaGen, blend bits, tcGen/tcMod |
| `textureBundle_t` | struct | Per-TMU image array, animation speed, tcGen, tcMod list, video handle |
| `texModInfo_t` | struct | One texture coordinate modifier (scroll, scale, rotate, turb, transform, stretch) |
| `deformStage_t` | struct | Vertex deformation descriptor (wave, bulge, move, normals, autosprite) |
| `waveForm_t` | struct | Waveform parameters: func, base, amplitude, phase, frequency |
| `collapse_t` | struct | Lookup table entry mapping two blend-mode pairs to a GL multitexture env mode |
| `infoParm_t` | struct | Maps surfaceParm keyword strings to `SURF_*`/`CONTENTS_*` flag bits |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_shaderText` | `char *` | static | Combined text of all loaded `.shader` files |
| `shader` | `shader_t` | static | Workspace for the shader being currently parsed |
| `stages` | `shaderStage_t[MAX_SHADER_STAGES]` | static | Workspace stages array for the shader being parsed |
| `texMods` | `texModInfo_t[MAX_SHADER_STAGES][TR_MAX_TEXMODS]` | static | Backing storage for `texMods` pointers during parsing |
| `deferLoad` | `qboolean` | static | Deferred-load flag (set but not used in this file) |
| `hashTable` | `shader_t*[1024]` | static | Hash table of all registered `shader_t` instances by name |
| `shaderTextHashTable` | `char**[2048]` | static | Hash table of text-block pointers for fast shader text lookup |

## Key Functions / Methods

### R_InitShaders
- **Signature:** `void R_InitShaders( void )`
- **Purpose:** Entry point to initialize the entire shader system at renderer startup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Clears `hashTable`, calls `CreateInternalShaders`, `ScanAndLoadShaderFiles`, `CreateExternalShaders`; populates `tr.defaultShader`, `tr.shadowShader`, `tr.projectionShadowShader`, `tr.flareShader`, `tr.sunShader`
- **Calls:** `CreateInternalShaders`, `ScanAndLoadShaderFiles`, `CreateExternalShaders`

### ScanAndLoadShaderFiles
- **Signature:** `static void ScanAndLoadShaderFiles( void )`
- **Purpose:** Scans `scripts/*.shader`, loads all files, concatenates them into `s_shaderText`, and builds `shaderTextHashTable` for O(1) name lookup.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Hunk-allocates `s_shaderText` and `shaderTextHashTable` memory; calls `ri.FS_ListFiles`, `ri.FS_ReadFile`, `ri.FS_FreeFile`
- **Calls:** `COM_ParseExt`, `SkipBracedSection`, `COM_Compress`, `generateHashValue`, `ri.Hunk_Alloc`
- **Notes:** Compresses whitespace/comments in each file buffer in-place before concatenation.

### R_FindShader
- **Signature:** `shader_t *R_FindShader( const char *name, int lightmapIndex, qboolean mipRawImage )`
- **Purpose:** Main shader lookup/creation routine. Returns existing shader or creates a new one (explicit from text or implicit from image).
- **Inputs:** Shader name, lightmap index constant or BSP lightmap index, mip flag
- **Outputs/Return:** Pointer to a valid `shader_t`; never NULL (falls back to `tr.defaultShader`)
- **Side effects:** Resets global `shader`/`stages` workspace; may upload images; calls `R_SyncRenderThread` if SMP; inserts into `hashTable` and `tr.shaders[]` via `FinishShader`
- **Calls:** `FindShaderInShaderText`, `ParseShader`, `FinishShader`, `R_FindImageFile`, `R_SyncRenderThread`

### ParseShader
- **Signature:** `static qboolean ParseShader( char **text )`
- **Purpose:** Parses the body of a shader text block into the global `shader`/`stages` workspace.
- **Inputs:** Pointer to current text parse position
- **Outputs/Return:** `qtrue` on success, `qfalse` on parse error
- **Side effects:** Writes to global `shader`, `stages`; sets `tr.sunLight`/`tr.sunDirection` for `q3map_sun`
- **Calls:** `ParseStage`, `ParseDeform`, `ParseSkyParms`, `ParseSurfaceParm`, `ParseSort`, `ParseVector`, `SkipRestOfLine`, `SkipBracedSection`, `COM_ParseExt`

### ParseStage
- **Signature:** `static qboolean ParseStage( shaderStage_t *stage, char **text )`
- **Purpose:** Parses one `{ }` stage block, filling a `shaderStage_t` with map, blend, rgbGen, alphaGen, tcGen, tcMod, and depth settings.
- **Inputs:** Target stage pointer, text position
- **Outputs/Return:** `qtrue`/`qfalse`
- **Side effects:** Calls `R_FindImageFile`; starts cinematic via `ri.CIN_PlayCinematic` for `videoMap`
- **Calls:** `COM_ParseExt`, `R_FindImageFile`, `ParseWaveForm`, `ParseVector`, `ParseTexMod`, `NameToSrcBlendMode`, `NameToDstBlendMode`, `NameToAFunc`

### FinishShader
- **Signature:** `static shader_t *FinishShader( void )`
- **Purpose:** Post-processes the global workspace shader (set sort, fog pass, detail culling, tcGen defaults, fog color adjustment), optionally collapses to vertex lighting or multitexture, then calls `GeneratePermanentShader`.
- **Inputs:** None (reads/writes global `shader`/`stages`)
- **Outputs/Return:** Pointer to the newly registered permanent `shader_t`
- **Side effects:** Modifies global `stages`; calls `VertexLightingCollapse`, `CollapseMultitexture`, `ComputeStageIteratorFunc`, `GeneratePermanentShader`

### GeneratePermanentShader
- **Signature:** `static shader_t *GeneratePermanentShader( void )`
- **Purpose:** Copies the global workspace shader to hunk memory, allocates stage pointers and texMod arrays, inserts into `tr.shaders[]`, sorts it via `SortNewShader`, and registers in `hashTable`.
- **Inputs:** None
- **Outputs/Return:** Pointer to the permanent hunk-allocated `shader_t`
- **Side effects:** Hunk allocates; increments `tr.numShaders`; calls `SortNewShader`

### CollapseMultitexture
- **Signature:** `static qboolean CollapseMultitexture( void )`
- **Purpose:** Attempts to merge `stages[0]` and `stages[1]` into a single multitexture pass using the `collapse[]` table.
- **Inputs:** None (operates on global `stages`)
- **Outputs/Return:** `qtrue` if collapsed
- **Side effects:** Modifies `stages[0]`, zeros `stages[MAX_SHADER_STAGES-1]`, sets `shader.multitextureEnv`
- **Notes:** Ensures lightmap is in bundle[1] for 3Dfx Voodoo compatibility.

### FixRenderCommandList
- **Signature:** `static void FixRenderCommandList( int newShader )`
- **Purpose:** Walks the current back-end render command list and increments `sortedIndex` for any draw surface whose index ≥ `newShader`, compensating for the insertion of a new shader into `tr.sortedShaders[]`.
- **Inputs:** Index of the newly inserted shader
- **Side effects:** Mutates `drawSurf_t.sort` fields in `backEndData[tr.smpFrame]->commands`

### R_RemapShader
- **Signature:** `void R_RemapShader( const char *shaderName, const char *newShaderName, const char *timeOffset )`
- **Purpose:** Redirects all shaders matching `shaderName` (across all lightmap variants) to render as `newShaderName`, optionally with a time offset.
- **Side effects:** Sets `sh->remappedShader` and `sh2->timeOffset`

### Notes (minor helpers)
- `generateHashValue` — djb-style hash over lowercase filename, strips extension, folds path separators.
- `NameToSrcBlendMode`, `NameToDstBlendMode`, `NameToAFunc`, `NameToGenFunc` — string-to-enum converters with warning fallbacks.
- `ParseWaveForm`, `ParseTexMod`, `ParseDeform`, `ParseSkyParms`, `ParseSurfaceParm`, `ParseSort` — sub-parsers called from `ParseShader`/`ParseStage`.

## Control Flow Notes
- **Init:** `R_InitShaders` is called during `R_Init` (renderer startup). It runs once per renderer initialization.
- **Runtime:** `R_FindShader` / `RE_RegisterShader*` are called during map loading and asset registration; results are cached in `hashTable`.
- **Frame:** No per-frame logic in this file directly; `FixRenderCommandList` is called at registration time (not per-frame) to patch in-flight command buffers.
- **Shutdown:** No explicit shutdown; hunk memory is freed by the engine hunk reset on renderer shutdown.

## External Dependencies
- `tr_local.h` — all renderer types (`shader_t`, `shaderStage_t`, `trGlobals_t tr`, `glConfig`, `ri`, cvars)
- `COM_ParseExt`, `SkipRestOfLine`, `SkipBracedSection`, `COM_Compress`, `COM_StripExtension`, `COM_DefaultExtension` — defined in `qcommon`
- `R_FindImageFile`, `R_InitSkyTexCoords` — defined in `tr_image.c` / `tr_sky.c`
- `RB_StageIteratorGeneric`, `RB_StageIteratorSky`, `RB_StageIteratorVertexLitTexture`, `RB_StageIteratorLightmappedMultitexture` — defined in `tr_shade.c`
- `R_SyncRenderThread` — defined in `tr_backend.c`
- `ri.Hunk_Alloc`, `ri.FS_ListFiles`, `ri.FS_ReadFile`, `ri.CIN_PlayCinematic` — engine import table (`refimport_t ri`)
- `backEndData`, `tr`, `glConfig` — renderer globals defined in `tr_init.c` / `tr_main.c`
