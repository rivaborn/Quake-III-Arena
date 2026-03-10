# code/server/sv_game.c

## File Purpose
This file implements the server-side interface between the Quake III engine and the game VM (virtual machine). It exposes engine services to the game DLL/bytecode through a system call dispatch table, and manages game VM lifecycle (init, restart, shutdown).

## Core Responsibilities
- Dispatch all game VM system calls via `SV_GameSystemCalls` (the single entry point for VM→engine calls)
- Translate between game-VM entity indices and server-side entity/client pointers
- Manage game VM lifecycle: load (`SV_InitGameProgs`), restart (`SV_RestartGameProgs`), shutdown (`SV_ShutdownGameProgs`)
- Forward bot library calls from the game VM to `botlib_export`
- Provide PVS (Potentially Visible Set) visibility tests for game logic
- Expose server state (serverinfo, userinfo, configstrings, usercmds) to the game VM

## Key Types / Data Structures
None defined locally; relies on types from `server.h` and `botlib.h`.

| Name | Kind | Purpose |
|---|---|---|
| `botlib_export_t` | typedef (struct) | Function table exported by the bot library; accessed via `botlib_export` global |
| `svEntity_t` | struct (defined in server.h) | Server-private entity data (world sector, cluster info, area nums) |
| `sharedEntity_t` | struct (defined in g_public.h) | Shared entity layout between server and game VM |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botlib_export` | `botlib_export_t *` | global | Pointer to the bot library's exported function table; set during bot init |

## Key Functions / Methods

### SV_GameSystemCalls
- **Signature:** `int SV_GameSystemCalls( int *args )`
- **Purpose:** Central dispatch function — the game VM calls into the engine through this single function pointer. Routes each `G_*`, `BOTLIB_*`, and `TRAP_*` opcode to the appropriate engine function.
- **Inputs:** `args` — integer array where `args[0]` is the syscall opcode and `args[1..n]` are typed arguments (pointers via `VMA()`, floats via `VMF()`)
- **Outputs/Return:** Integer result (often 0, or a meaningful value for query calls; floats returned via `FloatAsInt`)
- **Side effects:** Broad — triggers cvar changes, filesystem I/O, entity link/unlink, client drops, bot AI state mutations, trace operations
- **Calls:** Nearly every major engine subsystem: `CM_*`, `SV_*`, `FS_*`, `Cvar_*`, `Cmd_*`, `botlib_export->aas/ea/ai.*`, `VM_ArgPtr`, math intrinsics
- **Notes:** `VMA(x)` and `VMF(x)` macros abstract VM pointer/float argument extraction. PPC Linux has a special-cased `VMA` definition. `BOTLIB_EA_ACTION` uses `break` instead of `return 0` — falls through to `return -1`.

### SV_InitGameProgs
- **Signature:** `void SV_InitGameProgs( void )`
- **Purpose:** Full game VM initialization on a normal map load. Creates the VM and triggers `GAME_INIT`.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets global `gvm`; reads `bot_enable` cvar; calls `VM_Create` and `SV_InitGameVM`
- **Calls:** `Cvar_Get`, `VM_Create`, `SV_InitGameVM`, `Com_Error`

### SV_ShutdownGameProgs
- **Signature:** `void SV_ShutdownGameProgs( void )`
- **Purpose:** Shuts down and frees the game VM. Called on every map change.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `GAME_SHUTDOWN` in VM, then frees `gvm` and sets it to `NULL`
- **Calls:** `VM_Call`, `VM_Free`

### SV_RestartGameProgs
- **Signature:** `void SV_RestartGameProgs( void )`
- **Purpose:** Hot-restarts the game VM without a full map reload (`map_restart` command).
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `GAME_SHUTDOWN` with `qtrue`, calls `VM_Restart`, re-initializes via `SV_InitGameVM`
- **Calls:** `VM_Call`, `VM_Restart`, `SV_InitGameVM`, `Com_Error`

### SV_InitGameVM *(static)*
- **Signature:** `static void SV_InitGameVM( qboolean restart )`
- **Purpose:** Common init path shared by full init and restart. Resets entity parse point, clears gentity pointers, and calls `GAME_INIT` in the VM.
- **Inputs:** `restart` — passed to `GAME_INIT` to distinguish restart from fresh load
- **Side effects:** Modifies `sv.entityParsePoint`, clears `svs.clients[i].gentity` for all clients
- **Calls:** `CM_EntityString`, `VM_Call`, `Com_Milliseconds`

### SV_LocateGameData
- **Signature:** `void SV_LocateGameData( sharedEntity_t *gEnts, int numGEntities, int sizeofGEntity_t, playerState_t *clients, int sizeofGameClient )`
- **Purpose:** Called by the game VM (via `G_LOCATE_GAME_DATA`) to register its entity and client arrays with the server.
- **Side effects:** Writes `sv.gentities`, `sv.gentitySize`, `sv.num_entities`, `sv.gameClients`, `sv.gameClientSize`

### SV_NumForGentity / SV_GentityNum / SV_GameClientNum
- **Notes:** Pointer-arithmetic helpers that account for variable-size game entity structs (game appends private data beyond the shared region). Used throughout the server to safely convert between indices and pointers.

### SV_inPVS / SV_inPVSIgnorePortals
- **Purpose:** Test if two world points are mutually visible via BSP PVS clusters. `SV_inPVS` also checks area portal connectivity (doors block sight); `SV_inPVSIgnorePortals` does not.
- **Calls:** `CM_PointLeafnum`, `CM_LeafCluster`, `CM_LeafArea`, `CM_ClusterPVS`, `CM_AreasConnected`

### SV_SetBrushModel
- **Purpose:** Assigns an inline BSP brush model to an entity, computing its bounds and linking it into the world.
- **Side effects:** Sets `ent->s.modelindex`, `ent->r.mins/maxs/bmodel/contents`, calls `SV_LinkEntity`
- **Notes:** Contains a `FIXME` comment about removing the automatic `SV_LinkEntity` call.

### SV_GameCommand
- **Signature:** `qboolean SV_GameCommand( void )`
- **Purpose:** Forwards the current console command to the game VM for handling.
- **Calls:** `VM_Call( gvm, GAME_CONSOLE_COMMAND )`

## Control Flow Notes
- **Init:** `SV_InitGameProgs` → `VM_Create("qagame", SV_GameSystemCalls, ...)` → `SV_InitGameVM` → VM calls `G_LOCATE_GAME_DATA` back into `SV_GameSystemCalls`
- **Per-frame:** The game VM calls back into the engine on every tick via `SV_GameSystemCalls`
- **Shutdown:** `SV_ShutdownGameProgs` called on each map change before the next `SV_InitGameProgs`
- `SV_GameSystemCalls` is the sole VM→engine gateway registered at VM creation time

## External Dependencies
- `server.h` — `svs`, `sv`, `gvm`, all server types and function declarations
- `../game/botlib.h` — `botlib_export_t`, all `BOTLIB_*` syscall constants
- **Defined elsewhere:** `VM_Create`, `VM_Call`, `VM_Free`, `VM_Restart`, `VM_ArgPtr`; all `CM_*` collision functions; `SV_LinkEntity`, `SV_UnlinkEntity`, `SV_Trace`, `SV_AreaEntities`; `SV_BotAllocateClient`, `SV_BotLibSetup`, `SV_BotGetSnapshotEntity`; `BotImport_DebugPolygonCreate/Delete`; `FS_*`, `Cvar_*`, `Cmd_*`, `Cbuf_*`, `Com_*`, `Sys_*`, `MatrixMultiply`, `AngleVectors`, `PerpendicularVector`
