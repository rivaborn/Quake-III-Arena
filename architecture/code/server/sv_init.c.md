# code/server/sv_init.c

## File Purpose
Handles server initialization, map spawning, and shutdown for Quake III Arena. It manages configstring/userinfo get/set operations, client array allocation, and the full lifecycle of loading a new map while transitioning connected clients into the new game state.

## Core Responsibilities
- Register and initialize all server-side cvars at engine startup (`SV_Init`)
- Manage configstring storage and reliable broadcast to connected clients (`SV_SetConfigstring`)
- Allocate/reallocate the `svs.clients` array on startup or `sv_maxclients` change
- Execute the full map spawn sequence: clear state, load BSP, init game VM, settle frames, create delta baselines
- Transition existing connected clients (human and bot) into the new level
- Send final disconnect messages to all clients on server shutdown

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `server_t` (`sv`) | struct (extern) | Per-map server state: configstrings, entities, game VM pointers |
| `serverStatic_t` (`svs`) | struct (extern) | Persistent across maps: client array, snapshot entity pool, challenges |
| `client_t` | struct | Per-client state including connection state, netchan, download, snapshot info |
| `serverState_t` | enum | `SS_DEAD` / `SS_LOADING` / `SS_GAME` — governs when configstring updates broadcast |
| `clientState_t` | enum | `CS_FREE` → `CS_ZOMBIE` → `CS_CONNECTED` → `CS_PRIMED` → `CS_ACTIVE` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sv` | `server_t` | global (extern) | Per-level server data, cleared each map load |
| `svs` | `serverStatic_t` | global (extern) | Persistent server state across map changes |
| `gvm` | `vm_t *` | global (extern) | Handle to the game VM (`qagame`) |
| `sv_maxclients` | `cvar_t *` | global (extern) | Maximum allowed clients; governs array sizing |

## Key Functions / Methods

### SV_SetConfigstring
- **Signature:** `void SV_SetConfigstring(int index, const char *val)`
- **Purpose:** Updates a configstring slot and reliably broadcasts the change to all primed/active clients.
- **Inputs:** `index` — configstring slot; `val` — new string value (NULL treated as "")
- **Outputs/Return:** void
- **Side effects:** Frees and reallocates `sv.configstrings[index]`; sends `cs`/`bcs0`/`bcs1`/`bcs2` server commands to clients
- **Calls:** `Z_Free`, `CopyString`, `SV_SendServerCommand`, `Q_strncpyz`
- **Notes:** Large strings (≥ `MAX_STRING_CHARS - 24`) are chunked using the `bcs0/bcs1/bcs2` protocol. No-op if value is unchanged. Only broadcasts during `SS_GAME` or while `sv.restarting`.

### SV_CreateBaseline
- **Signature:** `void SV_CreateBaseline(void)`
- **Purpose:** Snapshots current entity states as delta-compression baselines to reduce initial sighting packet size.
- **Inputs:** None (reads `sv.svEntities`, `sv.num_entities`)
- **Outputs/Return:** void
- **Side effects:** Writes `sv.svEntities[entnum].baseline` for every linked entity
- **Calls:** `SV_GentityNum`
- **Notes:** Skips entity 0 (world); only baselines linked entities.

### SV_Startup
- **Signature:** `void SV_Startup(void)`
- **Purpose:** One-time allocation of `svs.clients` and snapshot entity pool when the server first goes live.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** `Z_Malloc` for `svs.clients`; sets `svs.numSnapshotEntities`; sets `sv_running` cvar to `"1"`
- **Calls:** `SV_BoundMaxClients`, `Z_Malloc`, `Cvar_Set`
- **Notes:** Fatal error if called when already initialized. Dedicated servers allocate `PACKET_BACKUP * 64` snapshot entities per client; listen servers allocate only `4 * 64`.

### SV_ChangeMaxClients
- **Signature:** `void SV_ChangeMaxClients(void)`
- **Purpose:** Resizes `svs.clients` when `sv_maxclients` is modified at runtime, preserving connected client state.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** `Z_Free` / `Z_Malloc` on `svs.clients`; uses `Hunk_AllocateTempMemory` as scratch buffer
- **Calls:** `SV_BoundMaxClients`, `Hunk_AllocateTempMemory`, `Hunk_FreeTempMemory`, `Z_Free`, `Z_Malloc`, `Com_Memset`
- **Notes:** Will never reduce capacity below the highest active client slot index.

### SV_SpawnServer
- **Signature:** `void SV_SpawnServer(char *server, qboolean killBots)`
- **Purpose:** Full map-change sequence — shuts down the old game, loads a new BSP, re-initializes the game VM, reconnects existing clients, and advances server state to `SS_GAME`.
- **Inputs:** `server` — map name (no path/extension); `killBots` — whether to drop bot clients
- **Outputs/Return:** void
- **Side effects:** Clears hunk, reloads filesystem with new checksum feed, loads collision map, runs 4 settling frames via game VM, sets numerous cvars (`mapname`, `sv_serverid`, `sv_paks`, etc.)
- **Calls:** `SV_ShutdownGameProgs`, `CL_MapLoading`, `CL_ShutdownAll`, `Hunk_Clear`, `CM_ClearMap`, `SV_Startup`/`SV_ChangeMaxClients`, `FS_Restart`, `CM_LoadMap`, `SV_ClearWorld`, `SV_InitGameProgs`, `VM_Call` (GAME_RUN_FRAME, GAME_CLIENT_CONNECT, GAME_CLIENT_BEGIN), `SV_BotFrame`, `SV_CreateBaseline`, `SV_DropClient`, `SV_Heartbeat_f`, `Hunk_SetMark`
- **Notes:** Not called for `map_restart` (that path goes through `SV_RestartGameProgs`). Sets `sv.state = SS_LOADING` before VM init to suppress premature configstring broadcasts, then flips to `SS_GAME` at the end.

### SV_Init
- **Signature:** `void SV_Init(void)`
- **Purpose:** Engine startup registration of all server cvars, operator commands, and bot library.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Registers ~30 cvars into global `cvar_t *` pointers; calls `SV_AddOperatorCommands`, `SV_BotInitCvars`, `SV_BotInitBotLib`
- **Calls:** `Cvar_Get`, `SV_AddOperatorCommands`, `SV_BotInitCvars`, `SV_BotInitBotLib`
- **Notes:** Called once at exe startup. `sv_pure` defaults differ between `DLL_ONLY` and VM builds.

### SV_Shutdown
- **Signature:** `void SV_Shutdown(char *finalmsg)`
- **Purpose:** Tears down the server — sends final disconnect to clients, frees all server memory, resets cvars.
- **Inputs:** `finalmsg` — string printed to clients before disconnect
- **Outputs/Return:** void
- **Side effects:** Calls `SV_FinalMessage`, clears `sv` and `svs`, sets `sv_running`/`ui_singlePlayerActive` to `"0"`, disconnects local client
- **Calls:** `SV_FinalMessage`, `SV_RemoveOperatorCommands`, `SV_MasterShutdown`, `SV_ShutdownGameProgs`, `SV_ClearServer`, `Z_Free`, `Com_Memset`, `Cvar_Set`, `CL_Disconnect`

## Control Flow Notes
- **Startup:** `SV_Init` → (map command) → `SV_SpawnServer` → `SV_Startup` (first time only)
- **Map change:** `SV_SpawnServer` handles the full reload; `map_restart` does NOT call this file's spawn path
- **Per-frame:** This file has no frame update logic; it is init/shutdown only
- **Shutdown:** `SV_Shutdown` called before `Sys_Quit` or `Sys_Error`

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `Z_Free`, `Z_Malloc`, `CopyString`, `Hunk_Alloc/Clear/SetMark`, `Hunk_AllocateTempMemory`, `VM_Call`, `VM_ExplicitArgPtr`, `FS_Restart`, `FS_ClearPakReferences`, `FS_LoadedPakChecksums/Names`, `FS_ReferencedPakChecksums/Names`, `CM_LoadMap`, `CM_ClearMap`, `CL_MapLoading`, `CL_ShutdownAll`, `CL_Disconnect`, `Cvar_Get/Set/VariableValue/InfoString/InfoString_Big`, `Com_Printf/Error/Milliseconds/Memset`, `SV_InitGameProgs`, `SV_ShutdownGameProgs`, `SV_ClearWorld`, `SV_SendServerCommand`, `SV_SendClientSnapshot`, `SV_DropClient`, `SV_BotFrame`, `SV_BotInitCvars`, `SV_BotInitBotLib`, `SV_Heartbeat_f`, `SV_MasterShutdown`, `SV_AddOperatorCommands`, `SV_RemoveOperatorCommands`
