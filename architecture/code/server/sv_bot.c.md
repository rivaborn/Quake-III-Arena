# code/server/sv_bot.c

## File Purpose
Serves as the server-side bridge between the Quake III game server and the BotLib AI library. It implements the `botlib_import_t` interface (callbacks the bot library calls into the engine) and exposes server-facing bot management functions for client slot allocation, per-frame ticking, and debug visualization.

## Core Responsibilities
- Allocate and free pseudo-client slots for bot entities
- Implement all `botlib_import_t` callbacks (trace, PVS, memory, file I/O, print, debug geometry)
- Initialize and populate the `botlib_import_t` vtable, then call `GetBotLibAPI` to obtain `botlib_export_t`
- Register all bot-related cvars at startup
- Drive the bot AI frame tick via `VM_Call(gvm, BOTAI_START_FRAME, time)`
- Provide bots access to reliable command queues and snapshot entity lists
- Manage a debug polygon pool for AAS visualization

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_debugpoly_t` | struct | Stores a single debug polygon (color, point count, up to 128 vec3 points) for AAS visualization |
| `botlib_import_t` | struct (extern, defined in botlib.h) | Vtable of engine callbacks provided to BotLib |
| `botlib_export_t` | struct (extern, defined in botlib.h) | Vtable of BotLib functions exposed to the server |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `debugpolygons` | `bot_debugpoly_t *` | file-static | Heap-allocated pool of debug polygons, size set by `bot_maxdebugpolys` |
| `bot_maxdebugpolys` | `int` | global | Maximum number of debug polygons; read from cvar at init |
| `botlib_export` | `botlib_export_t *` | global (extern) | Handle to BotLib's exported function table |
| `bot_enable` | `int` | global | Master enable flag; disables all bot processing when 0 |

## Key Functions / Methods

### SV_BotAllocateClient
- **Signature:** `int SV_BotAllocateClient(void)`
- **Purpose:** Finds a free `client_t` slot and configures it as an active bot client.
- **Inputs:** None
- **Outputs/Return:** Client index on success; `-1` if no free slot.
- **Side effects:** Mutates `svs.clients[i]` — sets state to `CS_ACTIVE`, address type to `NA_BOT`, rate, and links to a gentity.
- **Calls:** `SV_GentityNum`
- **Notes:** Does not call any game VM notification; caller must handle that.

### SV_BotFreeClient
- **Signature:** `void SV_BotFreeClient(int clientNum)`
- **Purpose:** Returns a bot's client slot to `CS_FREE` and clears the `SVF_BOT` flag on its gentity.
- **Inputs:** `clientNum` — index into `svs.clients`
- **Outputs/Return:** void
- **Side effects:** Modifies `svs.clients[clientNum]` and `cl->gentity->r.svFlags`.
- **Calls:** `Com_Error` on out-of-range input.

### BotImport_Trace
- **Signature:** `void BotImport_Trace(bsp_trace_t *bsptrace, vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int passent, int contentmask)`
- **Purpose:** Wraps `SV_Trace` and converts the result from `trace_t` to `bsp_trace_t` for BotLib.
- **Inputs:** Standard box-trace parameters.
- **Outputs/Return:** Fills `*bsptrace`; `exp_dist`, `sidenum`, `contents` are zeroed (not available from server trace).
- **Side effects:** None beyond the read-only collision query.
- **Calls:** `SV_Trace`

### BotImport_EntityTrace
- **Signature:** `void BotImport_EntityTrace(bsp_trace_t *bsptrace, vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int entnum, int contentmask)`
- **Purpose:** Same as `BotImport_Trace` but clips against a single specific entity.
- **Calls:** `SV_ClipToEntity`

### SV_BotInitBotLib
- **Signature:** `void SV_BotInitBotLib(void)`
- **Purpose:** Allocates the debug polygon pool, fills in the `botlib_import_t` vtable with all engine callbacks, and calls `GetBotLibAPI` to obtain and store `botlib_export`.
- **Inputs:** None
- **Outputs/Return:** void; sets global `botlib_export`.
- **Side effects:** Frees and reallocates `debugpolygons` (Zone), calls `GetBotLibAPI`.
- **Calls:** `Z_Free`, `Z_Malloc`, `Cvar_VariableIntegerValue`, `GetBotLibAPI`, `Sys_CheckCD`, `Com_Error`
- **Notes:** Asserts `botlib_export != NULL` after init.

### SV_BotFrame
- **Signature:** `void SV_BotFrame(int time)`
- **Purpose:** Per-server-frame entry point; triggers bot AI thinking via the game VM.
- **Inputs:** `time` — current server time in ms.
- **Side effects:** Calls into `gvm` via `VM_Call(gvm, BOTAI_START_FRAME, time)`.
- **Notes:** Returns immediately if `bot_enable == 0` or `gvm == NULL`.

### SV_BotGetConsoleMessage
- **Signature:** `int SV_BotGetConsoleMessage(int client, char *buf, int size)`
- **Purpose:** Dequeues one reliable server command for a bot to read as a console message.
- **Outputs/Return:** `qtrue` if a message was copied; `qfalse` if queue is empty or command slot is empty.
- **Side effects:** Increments `cl->reliableAcknowledge`.

### SV_BotGetSnapshotEntity
- **Signature:** `int SV_BotGetSnapshotEntity(int client, int sequence)`
- **Purpose:** Returns the entity number at a given index in the bot's current snapshot frame.
- **Outputs/Return:** Entity number or `-1` if `sequence` is out of range.

### BotDrawDebugPolygons
- **Signature:** `void BotDrawDebugPolygons(void (*drawPoly)(int,int,float*), int value)`
- **Purpose:** Iterates live debug polygons and dispatches them to a renderer callback; optionally runs BotLib's `Test` visualizer.
- **Side effects:** May call `botlib_export->BotLibVarSet` and `botlib_export->Test`; reads `svs.clients[0]` state (single-player debug assumption).

- **Notes:** `BotImport_DebugPolygonCreate/Show/Delete` and `BotImport_DebugLineCreate/Delete/Show` are helper functions managing the `debugpolygons` pool; line helpers are thin wrappers over the polygon variants.
- `SV_BotInitCvars` registers ~25 bot cvars and is called during server init.
- `SV_BotLibSetup` / `SV_BotLibShutdown` delegate directly to `botlib_export->BotLibSetup/Shutdown`.

## Control Flow Notes
- **Init:** `SV_BotInitCvars` → `SV_BotInitBotLib` (called from `SV_SpawnServer`/server init).
- **Per-frame:** `SV_BotFrame(time)` is called each server frame from `SV_Frame`, which dispatches into the game VM.
- **Shutdown:** `SV_BotLibShutdown` called on server kill or game directory change.
- Bot client slots participate in normal snapshot and command processing via the same `svs.clients` array as human clients.

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- `botlib.h` — defines `botlib_import_t`, `botlib_export_t`, `bsp_trace_t`, `bot_input_t`
- **Defined elsewhere:** `SV_Trace`, `SV_ClipToEntity`, `SV_PointContents`, `SV_inPVS`, `SV_ExecuteClientCommand`, `SV_GentityNum`, `CM_EntityString`, `CM_InlineModel`, `CM_ModelBounds`, `RadiusFromBounds`, `Z_TagMalloc`, `Z_Free`, `Z_Malloc`, `Z_AvailableMemory`, `Hunk_Alloc`, `Hunk_CheckMark`, `VM_Call`, `GetBotLibAPI`, `Sys_CheckCD`, `Cvar_Get`, `Cvar_VariableIntegerValue`, `Cvar_VariableValue`, `FS_FOpenFileByMode`, `FS_Read2`, `FS_Write`, `FS_FCloseFile`, `FS_Seek`, `gvm` (game VM handle), `svs`/`sv` server state globals.
