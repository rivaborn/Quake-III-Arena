# code/botlib/be_aas_main.c

## File Purpose
This is the main AAS (Area Awareness System) subsystem coordinator for Quake III's bot library. It manages the lifecycle of the AAS world — initialization, per-frame updates, map loading, and shutdown — and provides utility functions for string/model index lookups within the AAS world state.

## Core Responsibilities
- Lifecycle management: setup, load, init, per-frame update, shutdown of the AAS world
- Map loading: orchestrates loading of BSP and AAS files on map change
- Deferred initialization: drives incremental reachability and routing computation across frames
- String/model index registry: bidirectional lookup between config string indices and model names
- Developer diagnostics: exposes routing cache, memory usage, and memory dump via lib vars
- Routing cache persistence: triggers save of routing cache to disk on demand

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_t` | struct (defined in `be_aas_def.h`) | Global AAS world state: loaded/initialized flags, entity array, config strings, timing, frame counter |
| `libvar_t` | struct | Bot library variable (name/string/value/modified), used for `saveroutingcache` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aasworld` | `aas_t` | global | Singleton holding all AAS world state |
| `saveroutingcache` | `libvar_t *` | file-global | Pointer to lib var; when set to 1, triggers routing cache write on next frame |

## Key Functions / Methods

### AAS_Error
- **Signature:** `void QDECL AAS_Error(char *fmt, ...)`
- **Purpose:** Formats and dispatches a fatal-level error message through the bot import interface.
- **Inputs:** printf-style format string and variadic args
- **Outputs/Return:** void
- **Side effects:** Calls `botimport.Print(PRT_FATAL, ...)` — may trigger engine-level error handling
- **Calls:** `vsprintf`, `botimport.Print`

### AAS_StringFromIndex / AAS_IndexFromString
- **Purpose:** Bidirectional lookup between integer indices and string pointers in a string table. Guards against uninitialized indexes, out-of-range, and NULL entries.
- **Notes:** Both require `aasworld.indexessetup == qtrue`; return empty string / 0 on failure.

### AAS_ModelFromIndex / AAS_IndexFromModel
- **Purpose:** Thin wrappers over the above two, scoped to `CS_MODELS` config string range (up to `MAX_MODELS`).

### AAS_UpdateStringIndexes
- **Signature:** `void AAS_UpdateStringIndexes(int numconfigstrings, char *configstrings[])`
- **Purpose:** Copies incoming config strings into `aasworld.configstrings`, then marks `indexessetup = qtrue`.
- **Side effects:** Allocates heap memory via `GetMemory` for each non-NULL config string; does NOT free old pointers (commented out).
- **Notes:** Memory leak risk on repeated calls; old strings are not freed.

### AAS_ContinueInit
- **Signature:** `void AAS_ContinueInit(float time)`
- **Purpose:** Drives incremental AAS initialization across multiple frames: reachability calculation → clustering → optional optimization and file write → routing init → mark initialized.
- **Inputs:** Current game time
- **Side effects:** May call `AAS_Optimize`, `AAS_WriteAASFile`, `AAS_InitRouting`, `AAS_SetInitialized`; writes to disk if `savefile` or `forcewrite` lib var is set.
- **Calls:** `AAS_ContinueInitReachability`, `AAS_InitClustering`, `AAS_Optimize`, `AAS_WriteAASFile`, `AAS_InitRouting`, `AAS_SetInitialized`, `LibVarGetValue`, `LibVarValue`

### AAS_StartFrame
- **Signature:** `int AAS_StartFrame(float time)`
- **Purpose:** Per-frame AAS tick: update world time, unlink/invalidate stale entities, continue deferred init, handle developer diagnostic lib vars, persist routing cache if requested.
- **Inputs:** Current game time (float)
- **Outputs/Return:** `BLERR_NOERROR`
- **Side effects:** Mutates `aasworld.time`, `aasworld.frameroutingupdates`, `aasworld.numframes`; conditionally calls `AAS_WriteRouteCache`, `AAS_RoutingInfo`, `PrintUsedMemorySize`, `PrintMemoryLabels`
- **Calls:** `AAS_UnlinkInvalidEntities`, `AAS_InvalidateEntities`, `AAS_ContinueInit`, `AAS_RoutingInfo`, `AAS_WriteRouteCache`, `LibVarGetValue`, `LibVarSet`, `PrintUsedMemorySize`, `PrintMemoryLabels`

### AAS_LoadFiles
- **Signature:** `int AAS_LoadFiles(const char *mapname)`
- **Purpose:** Loads BSP info and the corresponding `.aas` file for a given map name.
- **Inputs:** Map name string (no extension)
- **Outputs/Return:** `BLERR_NOERROR` or error code from `AAS_LoadAASFile`
- **Side effects:** Sets `aasworld.mapname`, `aasworld.filename`; calls `AAS_ResetEntityLinks`, `AAS_LoadBSPFile`, `AAS_LoadAASFile`

### AAS_LoadMap
- **Signature:** `int AAS_LoadMap(const char *mapname)`
- **Purpose:** Full map change handler: frees old routing caches, loads new files, and initializes all AAS subsystems for the new map.
- **Inputs:** Map name (NULL means only update string indexes, returns 0 immediately)
- **Outputs/Return:** 0 on success, error code on failure
- **Side effects:** Sets `aasworld.initialized = qfalse`, `aasworld.loaded`; calls init functions for settings, link heap, linked entities, reachability, and alternative routing

### AAS_Setup
- **Signature:** `int AAS_Setup(void)`
- **Purpose:** First-time library setup: reads max clients/entities from lib vars, allocates entity array, invalidates all entities.
- **Side effects:** Allocates hunk memory for `aasworld.entities`; frees old allocation if present; resets `aasworld.numframes`
- **Calls:** `LibVarValue`, `LibVar`, `FreeMemory`, `GetClearedHunkMemory`, `AAS_InvalidateEntities`

### AAS_Shutdown
- **Signature:** `void AAS_Shutdown(void)`
- **Purpose:** Full AAS teardown: frees all subsystem data, entity memory, clears the `aasworld` struct.
- **Side effects:** Calls shutdown/free functions for alternative routing, BSP data, routing caches, link heap, linked entities, AAS data; zeroes `aasworld` via `Com_Memset`
- **Notes:** `aasworld.initialized` is explicitly set to `qfalse` after memset (redundant but defensive).

### AAS_ProjectPointOntoVector
- **Signature:** `void AAS_ProjectPointOntoVector(vec3_t point, vec3_t vStart, vec3_t vEnd, vec3_t vProj)`
- **Purpose:** Projects a 3D point onto a line segment defined by vStart→vEnd, writing the result to vProj.
- **Notes:** Does not clamp to segment endpoints; pure geometric utility.

## Control Flow Notes
- **Init:** `AAS_Setup` → `AAS_LoadMap` (on map change) → `AAS_StartFrame` (each frame, drives `AAS_ContinueInit` until fully initialized)
- **Per-frame:** `AAS_StartFrame` is the single engine-facing entry point called once per server frame from the bot library interface layer (`be_interface.c`)
- **Shutdown:** `AAS_Shutdown` called when the bot library is unloaded

## External Dependencies
- `q_shared.h`: `vec3_t`, `qboolean`, `Com_Memset`, `Com_sprintf`, `Q_stricmp`, `VectorSubtract`, `VectorNormalize`, `VectorMA`, `DotProduct`
- `l_memory.h`: `GetMemory`, `GetClearedHunkMemory`, `FreeMemory`, `PrintUsedMemorySize`, `PrintMemoryLabels`
- `l_libvar.h`: `LibVar`, `LibVarValue`, `LibVarGetValue`, `LibVarSet`
- `be_aas_def.h`: `aas_t`, `aas_entity_t` struct definitions (defined elsewhere)
- `be_interface.h`: `botimport` (defined elsewhere) — engine callback table for printing, file I/O, etc.; `bot_developer` flag
- Subsystem functions (defined elsewhere): `AAS_ContinueInitReachability`, `AAS_InitClustering`, `AAS_Optimize`, `AAS_WriteAASFile`, `AAS_InitRouting`, `AAS_UnlinkInvalidEntities`, `AAS_InvalidateEntities`, `AAS_ResetEntityLinks`, `AAS_LoadBSPFile`, `AAS_LoadAASFile`, `AAS_DumpBSPData`, `AAS_FreeRoutingCaches`, `AAS_InitAASLinkHeap`, `AAS_FreeAASLinkHeap`, `AAS_InitAASLinkedEntities`, `AAS_FreeAASLinkedEntities`, `AAS_DumpAASData`, `AAS_InitReachability`, `AAS_InitAlternativeRouting`, `AAS_ShutdownAlternativeRouting`, `AAS_InitSettings`, `AAS_WriteRouteCache`, `AAS_RoutingInfo`
