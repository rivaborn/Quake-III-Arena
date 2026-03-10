# code/bspc/be_aas_bspc.c

## File Purpose
This file is a BSPC (BSP Compiler) build-context adapter that provides stub implementations and wrappers for the botlib AAS (Area Awareness System) import interface. It bridges the botlib's `botlib_import_t` function table to the collision model (`CM_*`) and logging systems used by the offline map compiler tool, replacing the live engine's implementations.

## Core Responsibilities
- Define and populate the `botlib_import_t` struct for use in the BSPC tool context (not the live game engine)
- Wrap `CM_*` collision functions (`CM_BoxTrace`, `CM_PointContents`, `CM_InlineModel`, etc.) for botlib consumption
- Provide no-op stubs for debug visualization functions (`AAS_DebugLine`, `AAS_ClearShownDebugLines`)
- Provide a minimal timing function (`Sys_MilliSeconds`) via `clock()`
- Redirect print/log output to stdout and the BSPC log file
- Provide stubs for `Com_Memset`, `Com_Memcpy`, and `COM_Compress` required by shared code
- Drive the full AAS reachability and cluster computation pipeline via `AAS_CalcReachAndClusters`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `botlib_import_t` | struct (typedef, defined in `botlib.h`) | Function pointer table the botlib calls back into the engine/tool for services |
| `quakefile_t` | struct (typedef) | Describes a Quake file (possibly inside a pak/pk3); passed to `AAS_CalcReachAndClusters` |
| `bsp_trace_t` | struct (defined elsewhere) | Botlib-facing trace result; populated from `trace_t` returned by `CM_BoxTrace` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botimport` | `botlib_import_t` | global | The botlib import table populated by `AAS_InitBotImport`; used by all botlib subsystems |
| `worldmodel` | `clipHandle_t` | global | Handle to the loaded world inline model (index 0); used in all CM trace/contents calls |
| `capsule_collision` | `qboolean` | global (extern) | Controls whether capsule or box collision is used in `CM_BoxTrace` |

## Key Functions / Methods

### AAS_Error
- **Signature:** `void AAS_Error(char *fmt, ...)`
- **Purpose:** AAS error handler; formats a message and calls the fatal `Error()` function
- **Inputs:** printf-style format string and variadic args
- **Outputs/Return:** None (noreturn via `Error`)
- **Side effects:** Calls `Error()` which terminates the process
- **Calls:** `vsprintf`, `Error`
- **Notes:** `Error` is declared extern; defined in the BSPC main/utility code

### Sys_MilliSeconds
- **Signature:** `int Sys_MilliSeconds(void)`
- **Purpose:** Returns elapsed milliseconds using the C standard `clock()` function
- **Inputs:** None
- **Outputs/Return:** `int` millisecond count
- **Side effects:** None
- **Calls:** `clock()`
- **Notes:** Low-resolution; suitable only for progress timing in the offline tool

### BotImport_Trace
- **Signature:** `void BotImport_Trace(bsp_trace_t *bsptrace, vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int passent, int contentmask)`
- **Purpose:** Translates a botlib trace request into a `CM_BoxTrace` call and maps the result fields into `bsp_trace_t`
- **Inputs:** Output pointer, start/end positions, AABB extents, entity to pass through (unused here), content mask
- **Outputs/Return:** Fills `*bsptrace` in-place
- **Side effects:** None beyond output struct
- **Calls:** `CM_BoxTrace`, `VectorCopy`
- **Notes:** `passent` is ignored; `exp_dist` and `sidenum` are hardcoded to 0

### BotImport_BSPModelMinsMaxsOrigin
- **Signature:** `void BotImport_BSPModelMinsMaxsOrigin(int modelnum, vec3_t angles, vec3_t outmins, vec3_t outmaxs, vec3_t origin)`
- **Purpose:** Returns the axis-aligned bounding box (expanded for rotation) and origin of a BSP inline model
- **Inputs:** Inline model index, rotation angles, optional output pointers
- **Outputs/Return:** Writes to `outmins`, `outmaxs`, `origin` if non-NULL
- **Side effects:** None
- **Calls:** `CM_InlineModel`, `CM_ModelBounds`, `RadiusFromBounds`, `VectorCopy`, `VectorClear`
- **Notes:** Bug present — after expanding `mins[i]`, the `maxs[i]` computation re-reads the already-mutated `mins[i]`

### AAS_InitBotImport
- **Signature:** `void AAS_InitBotImport(void)`
- **Purpose:** Populates the global `botimport` function table with BSPC-context implementations
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes all relevant fields of the global `botimport` struct
- **Calls:** (assigns) `BotImport_BSPEntityData`, `BotImport_GetMemory`, `FreeMemory`, `BotImport_Trace`, `BotImport_PointContents`, `BotImport_Print`, `BotImport_BSPModelMinsMaxsOrigin`

### AAS_CalcReachAndClusters
- **Signature:** `void AAS_CalcReachAndClusters(struct quakefile_s *qf)`
- **Purpose:** Top-level pipeline entry point: loads the BSP collision map, initializes AAS subsystems, computes all reachabilities, then clusters
- **Inputs:** `qf` — Quake file descriptor for the map to process
- **Outputs/Return:** None (results written into `aasworld` global)
- **Side effects:** Loads collision map into CM system, modifies global `aasworld`, allocates AAS link heap
- **Calls:** `CM_LoadMap`, `CM_InlineModel`, `AAS_InitBotImport`, `AAS_LoadBSPFile`, `AAS_InitSettings`, `AAS_InitAASLinkHeap`, `AAS_InitAASLinkedEntities`, `AAS_SetViewPortalsAsClusterPortals`, `AAS_InitReachability`, `AAS_ContinueInitReachability`, `AAS_InitClustering`, `Log_Print`
- **Notes:** The reachability loop passes an ever-incrementing dummy `time` float; actual time semantics are internal to `AAS_ContinueInitReachability`

## Control Flow Notes
This file is not part of the live game frame loop. It is compiled into the **BSPC offline tool**. `AAS_CalcReachAndClusters` is called once during tool execution after a map is selected for processing. It drives the full AAS build pipeline sequentially to completion.

## External Dependencies
- `../game/q_shared.h` — shared math, types, trace structs
- `../bspc/l_log.h` — `Log_Print`, `Log_Write`
- `../bspc/l_qfiles.h` — `quakefile_t`
- `../botlib/l_memory.h` — `GetMemory`, `FreeMemory`
- `../qcommon/cm_public.h` — `CM_LoadMap`, `CM_BoxTrace`, `CM_PointContents`, `CM_InlineModel`, `CM_ModelBounds`, `CM_EntityString`
- `../botlib/be_aas_def.h` — `aasworld` global (defined elsewhere)
- `Error` — declared extern; defined in BSPC utility code
- `AAS_LoadBSPFile`, `AAS_InitSettings`, `AAS_InitAASLinkHeap`, `AAS_InitAASLinkedEntities`, `AAS_SetViewPortalsAsClusterPortals`, `AAS_InitReachability`, `AAS_ContinueInitReachability`, `AAS_InitClustering` — all defined in other botlib/AAS source files
