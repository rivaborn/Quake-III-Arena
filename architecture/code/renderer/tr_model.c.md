# code/renderer/tr_model.c

## File Purpose
Handles loading, caching, and querying of 3D models for the Quake III renderer. Supports two model formats: MD3 (static mesh with per-frame vertex animation) and MD4 (skeletal/bone-weighted mesh). Also provides tag interpolation and model bounds queries used during entity rendering.

## Core Responsibilities
- Allocate and register models into the global `tr.models[]` registry via handle
- Load and byte-swap MD3 files, including multi-LOD variants (up to `MD3_MAX_LODS`)
- Load and byte-swap MD4 files, including their LOD surfaces and bone-weighted vertices
- Register shaders referenced by model surfaces during load
- Interpolate MD3 attachment tags between animation frames (`R_LerpTag`)
- Provide model AABB bounds for culling (`R_ModelBounds`)
- Initialize the model subsystem and expose a debug listing command

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `model_t` | struct | Per-model record: name, type, index, LOD md3 headers, md4 header, bmodel pointer |
| `modtype_t` | enum | `MOD_BAD`, `MOD_BRUSH`, `MOD_MESH`, `MOD_MD4` — discriminates union-like fields in `model_t` |
| `md3Header_t` | struct | On-disk/in-memory MD3 file header; owns frame, tag, and surface sub-arrays |
| `md4Header_t` | struct | On-disk/in-memory MD4 file header; owns frames, bones, and LOD sub-arrays |
| `md3Tag_t` | struct | Named attachment point with origin + 3-axis orientation, per animation frame |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `loadmodel` | `model_t *` | global (file-level) | Points to the model currently being loaded; used by loader helpers |

## Key Functions / Methods

### R_GetModelByHandle
- **Signature:** `model_t *R_GetModelByHandle( qhandle_t index )`
- **Purpose:** Resolve a renderer model handle to its `model_t *`.
- **Inputs:** `index` — opaque handle (1-based index into `tr.models[]`)
- **Outputs/Return:** Pointer to `model_t`; returns `tr.models[0]` (null/default model) for out-of-range handles.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Handle 0 and out-of-range are treated identically as the default model.

---

### R_AllocModel
- **Signature:** `model_t *R_AllocModel( void )`
- **Purpose:** Allocate a new `model_t` slot in `tr.models[]` from the hunk.
- **Inputs:** None.
- **Outputs/Return:** Pointer to new `model_t`, or `NULL` if `MAX_MOD_KNOWN` reached.
- **Side effects:** Increments `tr.numModels`; hunk-allocates memory (`h_low`).
- **Calls:** `ri.Hunk_Alloc`
- **Notes:** Index 0 is reserved as the null/bad model (allocated in `R_ModelInit`).

---

### RE_RegisterModel
- **Signature:** `qhandle_t RE_RegisterModel( const char *name )`
- **Purpose:** Public entry point to load or look up a named model; returns a handle usable by the rest of the engine.
- **Inputs:** `name` — virtual filesystem path (e.g., `models/players/keel/upper.md3`).
- **Outputs/Return:** Non-zero `qhandle_t` on success; `0` on failure.
- **Side effects:** Allocates `model_t`, reads files via `ri.FS_ReadFile`, hunk-allocates model data, registers shaders. On failure, marks model `MOD_BAD` and retains the slot to avoid future rescanning.
- **Calls:** `R_AllocModel`, `R_SyncRenderThread`, `ri.FS_ReadFile`, `R_LoadMD3`, `R_LoadMD4`, `ri.FS_FreeFile`
- **Notes:** Iterates LODs from highest (`MD3_MAX_LODS-1`) down to 0; appended `_1.md3`, `_2.md3` suffixes for non-base LODs. Successfully loaded lower-detail LODs are mirrored up into unused higher-detail slots for runtime `r_lodbias` changes.

---

### R_LoadMD3
- **Signature:** `static qboolean R_LoadMD3( model_t *mod, int lod, void *buffer, const char *mod_name )`
- **Purpose:** Parse, validate, hunk-copy, and byte-swap a raw MD3 buffer into `mod->md3[lod]`.
- **Inputs:** `mod` — target model; `lod` — LOD slot index; `buffer` — raw file data; `mod_name` — for error messages.
- **Outputs/Return:** `qtrue` on success, `qfalse` on version mismatch or empty frame count.
- **Side effects:** Hunk-allocates model data; increments `mod->dataSize`; calls `R_FindShader` for each surface shader; sets `surf->ident = SF_MD3`; lowercases surface names; strips trailing `_1`/`_2` suffixes from surface names.
- **Calls:** `ri.Hunk_Alloc`, `Com_Memcpy`, `R_FindShader`, `ri.Error`, `Q_strlwr`
- **Notes:** Validates `surf->numVerts <= SHADER_MAX_VERTEXES` and `surf->numTriangles*3 <= SHADER_MAX_INDEXES`; errors on excess with `ERR_DROP`. XyzNormals are stored as `short` and swapped with `LittleShort`.

---

### R_LoadMD4
- **Signature:** `static qboolean R_LoadMD4( model_t *mod, void *buffer, const char *mod_name )`
- **Purpose:** Parse, validate, hunk-copy, and byte-swap a raw MD4 skeletal model buffer.
- **Inputs:** Same pattern as `R_LoadMD3` minus `lod`.
- **Outputs/Return:** `qtrue` on success, `qfalse` on version mismatch or empty frame count.
- **Side effects:** Hunk-allocates; sets `mod->type = MOD_MD4`; swaps all bone matrix floats, vertex weights, and triangle indices; registers per-surface shaders.
- **Calls:** `ri.Hunk_Alloc`, `Com_Memcpy`, `R_FindShader`, `ri.Error`, `Q_strlwr`
- **Notes:** Bug present — `frame` pointer is used before assignment in the frame-swap loop (declared uninitialized, then assigned inside the loop body). Vertex stride is variable due to `numWeights`; pointer is manually advanced with `&v->weights[v->numWeights]`.

---

### RE_BeginRegistration
- **Signature:** `void RE_BeginRegistration( glconfig_t *glconfigOut )`
- **Purpose:** Start a new content registration phase; initializes renderer and resets scene state.
- **Inputs:** `glconfigOut` — output struct to receive current GL configuration.
- **Side effects:** Calls `R_Init`, resets `tr.viewCluster`, clears flares and scene, sets `tr.registered = qtrue`, issues a zero-size `RE_StretchPic` workaround for first-frame flash.
- **Calls:** `R_Init`, `R_SyncRenderThread`, `R_ClearFlares`, `RE_ClearScene`, `RE_StretchPic`

---

### R_LerpTag
- **Signature:** `int R_LerpTag( orientation_t *tag, qhandle_t handle, int startFrame, int endFrame, float frac, const char *tagName )`
- **Purpose:** Linearly interpolate an MD3 attachment tag's origin and axes between two frames.
- **Inputs:** Model handle, start/end frame indices, blend fraction `frac` (0=start, 1=end), tag name.
- **Outputs/Return:** `qtrue` if tag found and interpolated; `qfalse` otherwise (tag/axis cleared).
- **Side effects:** Writes to `*tag`; normalizes all three axes after interpolation.
- **Calls:** `R_GetModelByHandle`, `R_GetTag`, `AxisClear`, `VectorClear`, `VectorNormalize`
- **Notes:** Only queries `md3[0]` (base LOD). Non-orthogonal intermediate results are renormalized per-axis independently (not re-orthogonalized).

---

### R_ModelBounds
- **Signature:** `void R_ModelBounds( qhandle_t handle, vec3_t mins, vec3_t maxs )`
- **Purpose:** Return the AABB for a model (frame 0 for MD3, bmodel bounds for brush models).
- **Side effects:** Writes to `mins`/`maxs`; clears both to zero if no geometry available.

## Control Flow Notes
- `R_ModelInit` is called during renderer init to seed the null model at slot 0.
- `RE_BeginRegistration` is the engine's signal to begin loading content; it drives `R_Init`.
- `RE_RegisterModel` is called by `cgame`/`game` modules per-asset during map load.
- During rendering, `R_LerpTag` and `R_ModelBounds` are called per-frame from entity rendering code (`tr_main.c`, `cg_ents.c` proxy through the renderer API).
- No per-frame update logic lives here; this file is load-time and query-time only.

## External Dependencies
- **`tr_local.h`** — pulls in all renderer types, `tr` globals, `ri` refimport, `qfiles.h` MD3/MD4 structs
- `ri.Hunk_Alloc`, `ri.FS_ReadFile`, `ri.FS_FreeFile`, `ri.Printf`, `ri.Error` — engine syscalls via `refimport_t ri`
- `R_FindShader` — defined in `tr_shader.c`
- `R_Init`, `R_SyncRenderThread`, `R_ClearFlares`, `RE_ClearScene`, `RE_StretchPic` — defined elsewhere in the renderer
- `Com_Memcpy`, `Q_strncpyz`, `Q_strlwr`, `VectorNormalize`, `AxisClear`, `VectorClear` — shared utilities from `q_shared.c`/`q_math.c`
