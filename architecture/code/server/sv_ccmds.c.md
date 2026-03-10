# code/server/sv_ccmds.c

## File Purpose
Implements operator/admin console commands for the Quake III Arena server. These commands are restricted to stdin or remote operator datagrams and cover server management: map loading, player kicking/banning, status reporting, and server lifecycle control.

## Core Responsibilities
- Register all server operator commands via `SV_AddOperatorCommands`
- Resolve clients by name or slot number for targeted operations
- Load and restart maps (including single-player, devmap, and warmup-delayed restarts)
- Kick and ban players by name or client number
- Print server status, serverinfo, systeminfo, and per-user info to the console
- Broadcast console chat messages to all connected clients
- Force the next heartbeat to fire immediately

## Key Types / Data Structures
None defined in this file; uses `client_t`, `playerState_t`, `server_t` (`sv`), `serverStatic_t` (`svs`) from `server.h`.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `initialized` (inside `SV_AddOperatorCommands`) | `static qboolean` | static/local | Guards one-time command registration |

## Key Functions / Methods

### SV_GetPlayerByName
- **Signature:** `static client_t *SV_GetPlayerByName(void)`
- **Purpose:** Resolves the connected client whose name matches `Cmd_Argv(1)`, with a fallback strip-color comparison.
- **Inputs:** Implicit — reads `Cmd_Argv(1)`, `svs.clients`, `sv_maxclients`
- **Outputs/Return:** Pointer to matching `client_t`, or `NULL`
- **Side effects:** Prints to console on failure
- **Calls:** `Cmd_Argc`, `Cmd_Argv`, `Q_stricmp`, `Q_strncpyz`, `Q_CleanStr`, `Com_Printf`
- **Notes:** Skips slots with `cl->state == 0` (CS_FREE)

### SV_GetPlayerByNum
- **Signature:** `static client_t *SV_GetPlayerByNum(void)`
- **Purpose:** Resolves a connected client by numeric slot index from `Cmd_Argv(1)`.
- **Inputs:** Implicit — reads `Cmd_Argv(1)`, `svs.clients`
- **Outputs/Return:** Pointer to `client_t`, or `NULL`
- **Side effects:** Console prints on bad input
- **Calls:** `Cmd_Argc`, `Cmd_Argv`, `atoi`, `Com_Printf`
- **Notes:** Contains an unreachable second `return NULL` (dead code bug)

### SV_Map_f
- **Signature:** `static void SV_Map_f(void)`
- **Purpose:** Loads a new map; handles `map`, `devmap`, `spmap`, `spdevmap` command variants. Sets gametype, maxclients, and cheat mode accordingly.
- **Inputs:** `Cmd_Argv(0)` (command name), `Cmd_Argv(1)` (map name)
- **Outputs/Return:** void
- **Side effects:** Calls `SV_SpawnServer`; sets `sv_cheats`, `g_gametype`, `g_doWarmup`, `sv_maxclients` cvars; validates map file exists via `FS_ReadFile`
- **Calls:** `Cmd_Argv`, `Com_sprintf`, `FS_ReadFile`, `Cvar_Get`, `Cvar_SetValue`, `Cvar_SetLatched`, `Cvar_Set`, `SV_SpawnServer`, `Q_stricmpn`, `Q_stricmp`, `Q_strncpyz`
- **Notes:** Map name is saved before `SV_SpawnServer` because the q3config.cfg reload would destroy `Cmd_Argv` state

### SV_MapRestart_f
- **Signature:** `static void SV_MapRestart_f(void)`
- **Purpose:** Restarts the current level in-place without sending a new gamestate, supporting optional warmup delay. Handles full respawn if key cvars changed.
- **Inputs:** Optional `Cmd_Argv(1)` delay in seconds
- **Outputs/Return:** void
- **Side effects:** Modifies `sv.serverId`, `sv.state`, `sv.restarting`, `svs.snapFlagServerBit`, `svs.time`; calls `SV_RestartGameProgs`; reconnects all clients via VM calls; may call `SV_SpawnServer` for full restart
- **Calls:** `SV_RestartGameProgs`, `VM_Call`, `VM_ExplicitArgPtr`, `SV_AddServerCommand`, `SV_DropClient`, `SV_ClientEnterWorld`, `SV_SetConfigstring`, `SV_SpawnServer`, `Cvar_Set`, `Cvar_VariableValue`, `Cvar_VariableString`
- **Notes:** Guards against double-restart in same frame via `com_frameTime == sv.serverId`; runs 3+1 settling frames after game VM restart

### SV_Kick_f / SV_KickNum_f
- **Signature:** `static void SV_Kick_f(void)` / `static void SV_KickNum_f(void)`
- **Purpose:** Drop a client by name or slot number. `SV_Kick_f` additionally supports `"all"` and `"allbots"` targets.
- **Side effects:** Calls `SV_DropClient`; sets `cl->lastPacketTime` to prevent zombie re-entry
- **Notes:** Loopback (host) player is protected from kicks

### SV_Ban_f / SV_BanNum_f
- **Signature:** `static void SV_Ban_f(void)` / `static void SV_BanNum_f(void)`
- **Purpose:** Submits a player's IP to the external authorize server for banning.
- **Side effects:** Resolves `AUTHORIZE_SERVER_NAME` via `NET_StringToAdr` on first use; sends OOB `banUser` UDP packet via `NET_OutOfBandPrint`
- **Notes:** Does not drop the client immediately; only contacts the auth server

### SV_AddOperatorCommands
- **Signature:** `void SV_AddOperatorCommands(void)`
- **Purpose:** Registers all operator console commands exactly once.
- **Side effects:** Calls `Cmd_AddCommand` for each command; conditionally registers `say` on dedicated servers and map variants outside `PRE_RELEASE_DEMO`
- **Notes:** `SV_RemoveOperatorCommands` is a no-op (body `#if 0`'d) with a comment explaining removal would break server restart

## Control Flow Notes
Called during server init (`sv_main.c`). `SV_AddOperatorCommands` runs once at startup. Individual command handlers execute in response to console or rcon input, outside the normal per-frame game loop. `SV_MapRestart_f` directly drives VM frames (`GAME_RUN_FRAME`) and client reconnect logic inline.

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `svs`, `sv`, `gvm`, `sv_maxclients`, `sv_gametype`, `sv_mapname`, `com_sv_running`, `com_frameTime`, `com_dedicated`; `SV_SpawnServer`, `SV_DropClient`, `SV_ClientEnterWorld`, `SV_RestartGameProgs`, `SV_AddServerCommand`, `SV_SendServerCommand`, `SV_SetConfigstring`, `SV_GameClientNum`, `SV_SectorList_f`; `VM_Call`, `VM_ExplicitArgPtr`; `NET_StringToAdr`, `NET_OutOfBandPrint`, `NET_AdrToString`; `Cmd_*`, `Cvar_*`, `Info_Print`, `FS_ReadFile`
