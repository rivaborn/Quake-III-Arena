# code/renderer/tr_public.h

## File Purpose
Defines the public ABI boundary between the Quake III renderer module and the engine/client. It declares two function-pointer structs (`refexport_t` and `refimport_t`) and the single DLL entry point `GetRefAPI`, enabling the renderer to be loaded as a dynamically swappable module.

## Core Responsibilities
- Define `REF_API_VERSION` for compatibility checking at load time
- Declare all renderer-exported functions via `refexport_t` (scene building, resource registration, frame control, etc.)
- Declare all engine services imported by the renderer via `refimport_t` (memory, filesystem, cvars, commands, etc.)
- Expose `GetRefAPI` as the sole linker-visible symbol for module initialization
- Include `tr_types.h` to bring shared render types into scope

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `refexport_t` | struct (function-pointer table) | All functions the renderer exposes to the engine/client |
| `refimport_t` | struct (function-pointer table) | All engine services the renderer calls back into |

## Global / File-Static State
None.

## Key Functions / Methods

### GetRefAPI
- **Signature:** `refexport_t* GetRefAPI( int apiVersion, refimport_t *rimp );`
- **Purpose:** Module entry point. Validates `apiVersion` against `REF_API_VERSION`, stores the import table, and returns a pointer to the filled `refexport_t`. Returns `NULL` if initialization fails.
- **Inputs:** `apiVersion` — caller's expected API version; `rimp` — engine-provided import table.
- **Outputs/Return:** Pointer to the renderer's `refexport_t`, or `NULL` on failure.
- **Side effects:** Initializes the renderer subsystem; stores `rimp` in renderer-internal globals (defined in `tr_init.c`).
- **Calls:** Defined in `code/renderer/tr_init.c`; not visible here.
- **Notes:** Only linker-exported symbol; all other renderer functions are accessed through the returned struct.

### refexport_t members (summary)

Key entry points exposed to the engine:

- **Registration phase:** `BeginRegistration`, `RegisterModel`, `RegisterSkin`, `RegisterShader`, `RegisterShaderNoMip`, `LoadWorld`, `SetWorldVisData`, `EndRegistration` — load and register all level assets before rendering begins.
- **Scene construction:** `ClearScene`, `AddRefEntityToScene`, `AddPolyToScene`, `AddLightToScene`, `AddAdditiveLightToScene` — called each frame to populate the scene list.
- **Frame control:** `BeginFrame` (stereo selection), `RenderScene` (executes the render), `EndFrame` (swap buffers; optionally returns front/back-end timing).
- **2D drawing:** `SetColor`, `DrawStretchPic`, `DrawStretchRaw`, `UploadCinematic` — HUD, cinematics, and UI rendering.
- **Queries:** `MarkFragments`, `LerpTag`, `ModelBounds`, `LightForPoint`, `inPVS`, `GetEntityToken`.
- **Misc:** `Shutdown`, `RegisterFont`, `RemapShader`.

### refimport_t members (summary)

Engine services consumed by the renderer:

- **Output/error:** `Printf`, `Error`.
- **Memory:** `Hunk_Alloc` / `Hunk_AllocDebug`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Malloc`, `Free`.
- **Cvars/commands:** `Cvar_Get`, `Cvar_Set`, `Cmd_AddCommand`, `Cmd_RemoveCommand`, `Cmd_Argc`, `Cmd_Argv`, `Cmd_ExecuteText`.
- **Filesystem:** `FS_FileIsInPAK`, `FS_ReadFile`, `FS_FreeFile`, `FS_ListFiles`, `FS_FreeFileList`, `FS_WriteFile`, `FS_FileExists`.
- **Cinematics:** `CIN_UploadCinematic`, `CIN_PlayCinematic`, `CIN_RunCinematic`.
- **Debug:** `CM_DrawDebugSurface`, `Milliseconds`.

## Control Flow Notes

This header defines the **initialization and per-frame contract** for the renderer module:

- **Init:** `GetRefAPI` is called once at startup (or on `vid_restart`). The engine passes `refimport_t`; the renderer returns `refexport_t`.
- **Per-level load:** `BeginRegistration` → `Register*` / `LoadWorld` → `EndRegistration`.
- **Per-frame:** `BeginFrame` → `ClearScene` → `Add*ToScene` (many calls) → `RenderScene` → optional 2D draws → `EndFrame`.
- **Shutdown:** `Shutdown(destroyWindow)` — `qfalse` for `vid_restart`, `qtrue` for full exit.

## External Dependencies

- **`../cgame/tr_types.h`** — shared render types: `refEntity_t`, `refdef_t`, `polyVert_t`, `glconfig_t`, `stereoFrame_t`, `markFragment_t`, `fontInfo_t`, `orientation_t`, `qhandle_t`, `refEntityType_t`, etc.
- **Defined elsewhere:** `vec3_t`, `qboolean`, `byte`, `cvar_t`, `ha_pref`, `e_status`, `QDECL`, `BIG_INFO_STRING`, `MAX_STRING_CHARS`, `MAX_MAP_AREA_BYTES` — all from `q_shared.h` or platform headers pulled in transitively.
- **`__USEA3D`** — conditional A3D audio-geometry hook; platform-specific, not cross-platform.
