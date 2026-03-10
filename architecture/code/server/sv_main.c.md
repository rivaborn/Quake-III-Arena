# code/server/sv_main.c

## File Purpose
Core server frame driver and network dispatch hub for Quake III Arena. It owns the two primary server-side globals (`svs`, `sv`, `gvm`), drives the per-frame game simulation loop, and routes all incoming UDP packets — both connectionless and in-sequence — to appropriate handlers.

## Core Responsibilities
- Define and expose all server-side cvars
- Manage reliable server-command queuing per client (`SV_AddServerCommand`, `SV_SendServerCommand`)
- Send/receive heartbeats to/from master servers
- Respond to connectionless queries: `getstatus`, `getinfo`, `getchallenge`, `connect`, `rcon`, `ipAuthorize`
- Dispatch sequenced in-game packets to the correct `client_t` via `SV_PacketEvent`
- Run the main server frame: ping calculation, timeout detection, game VM tick, snapshot dispatch, heartbeat

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `serverStatic_t` | struct (typedef) | Persistent server state across map changes (clients, challenges, snapshot entity pool) |
| `server_t` | struct (typedef) | Per-map server state (configstrings, entities, game VM pointers, restart time) |
| `client_t` | struct (typedef) | Per-connection client state (reliable command ring, download state, ping frames, netchan) |
| `clientSnapshot_t` | struct (typedef) | One stored snapshot frame used for delta compression and RTT measurement |
| `clientState_t` | enum | Client lifecycle: `CS_FREE` → `CS_ZOMBIE` → `CS_CONNECTED` → `CS_PRIMED` → `CS_ACTIVE` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `svs` | `serverStatic_t` | global | Persistent server info (clients array, challenges, snapshot entities) |
| `sv` | `server_t` | global | Current map's server state |
| `gvm` | `vm_t *` | global | Handle to the game logic VM |
| `sv_fps`, `sv_timeout`, `sv_zombietime`, `sv_rconPassword`, `sv_privatePassword`, `sv_allowDownload`, `sv_maxclients`, `sv_privateClients`, `sv_hostname`, `sv_master[5]`, `sv_reconnectlimit`, `sv_showloss`, `sv_padPackets`, `sv_killserver`, `sv_mapname`, `sv_mapChecksum`, `sv_serverid`, `sv_maxRate`, `sv_minPing`, `sv_maxPing`, `sv_gametype`, `sv_pure`, `sv_floodProtect`, `sv_lanForceRate`, `sv_strictAuth` | `cvar_t *` | global | All server-side configurable variables |

## Key Functions / Methods

### SV_AddServerCommand
- **Signature:** `void SV_AddServerCommand(client_t *client, const char *cmd)`
- **Purpose:** Appends a reliable command into the client's circular ring buffer; drops the client if the ring overflows (unacknowledged commands exceed `MAX_RELIABLE_COMMANDS`).
- **Inputs:** `client` — target client; `cmd` — null-terminated command string
- **Outputs/Return:** void
- **Side effects:** Mutates `client->reliableSequence` and `client->reliableCommands[]`; may call `SV_DropClient`
- **Calls:** `SV_DropClient`, `Q_strncpyz`, `Com_Printf`
- **Notes:** Uses `==` not `>=` on overflow check intentionally to avoid recursive drop from broadcast prints.

### SV_SendServerCommand
- **Signature:** `void QDECL SV_SendServerCommand(client_t *cl, const char *fmt, ...)`
- **Purpose:** Variadic printf-style broadcast or unicast of a reliable server command; echoes `print` commands to the dedicated console.
- **Inputs:** `cl` — specific client or NULL for broadcast; `fmt` — format string
- **Outputs/Return:** void
- **Side effects:** Calls `SV_AddServerCommand` for one or all primed+ clients
- **Calls:** `SV_AddServerCommand`, `Com_Printf`, `SV_ExpandNewlines`
- **Notes:** Skips clients below `CS_PRIMED` on broadcast.

### SV_MasterHeartbeat
- **Signature:** `void SV_MasterHeartbeat(void)`
- **Purpose:** Sends OOB `heartbeat QuakeArena-1` UDP packets to all configured master servers; resolves hostnames lazily.
- **Inputs:** None (reads `com_dedicated`, `svs.time`, `sv_master[]`)
- **Outputs/Return:** void
- **Side effects:** Mutates `svs.nextHeartbeatTime`; performs DNS resolution via `NET_StringToAdr`; clears invalid master cvar strings
- **Calls:** `NET_StringToAdr`, `Cvar_Set`, `NET_OutOfBandPrint`, `BigShort`, `Com_Printf`
- **Notes:** Only fires for `com_dedicated == 2` (internet-public); 5-minute interval (`HEARTBEAT_MSEC`).

### SVC_Status
- **Signature:** `void SVC_Status(netadr_t from)`
- **Purpose:** Responds to a `getstatus` OOB query with full server info and connected player list.
- **Inputs:** `from` — sender address
- **Side effects:** Sends UDP reply via `NET_OutOfBandPrint`
- **Calls:** `Cvar_InfoString`, `Info_SetValueForKey`, `SV_GameClientNum`, `NET_OutOfBandPrint`
- **Notes:** Silent in single-player mode; echoes challenge token to prevent spoofed ghost-server injection.

### SVC_Info
- **Signature:** `void SVC_Info(netadr_t from)`
- **Purpose:** Responds to `getinfo` with a compact infostring (hostname, map, player count, gametype, pure flag).
- **Calls:** `Info_SetValueForKey`, `NET_OutOfBandPrint`, `Cvar_VariableValue`, `Cvar_VariableString`

### SVC_RemoteCommand
- **Signature:** `void SVC_RemoteCommand(netadr_t from, msg_t *msg)`
- **Purpose:** Authenticates and executes an rcon command, redirecting all `Com_Printf` output back to the sender as OOB print packets.
- **Side effects:** 500 ms rate-limit via `static lasttime`; calls `Com_BeginRedirect`/`Com_EndRedirect`; executes arbitrary console commands on auth
- **Calls:** `Com_BeginRedirect`, `Com_EndRedirect`, `Cmd_ExecuteString`, `Com_Milliseconds`

### SV_ConnectionlessPacket
- **Signature:** `void SV_ConnectionlessPacket(netadr_t from, msg_t *msg)`
- **Purpose:** Top-level dispatcher for OOB packets; Huffman-decompresses `connect` packets before tokenizing.
- **Calls:** `SVC_Status`, `SVC_Info`, `SV_GetChallenge`, `SV_DirectConnect`, `SV_AuthorizeIpPacket`, `SVC_RemoteCommand`, `Huff_Decompress`

### SV_PacketEvent
- **Signature:** `void SV_PacketEvent(netadr_t from, msg_t *msg)`
- **Purpose:** Routes a received UDP packet — OOB to `SV_ConnectionlessPacket`, or sequenced to the matching `client_t` identified by base IP + qport.
- **Side effects:** May correct a NAT-translated port; calls `SV_ExecuteClientMessage` for active clients; sends OOB `disconnect` for unknown addresses
- **Calls:** `SV_ConnectionlessPacket`, `SV_Netchan_Process`, `SV_ExecuteClientMessage`, `NET_OutOfBandPrint`

### SV_CalcPings
- **Signature:** `void SV_CalcPings(void)`
- **Purpose:** Recomputes each active client's ping as the average of `messageAcked - messageSent` across all `PACKET_BACKUP` frames; writes result into `playerState_t.ping`.
- **Calls:** `SV_GameClientNum`

### SV_CheckTimeouts
- **Signature:** `void SV_CheckTimeouts(void)`
- **Purpose:** Transitions zombie clients to `CS_FREE` and drops unresponsive active clients after 5 consecutive timeout frames.
- **Calls:** `SV_DropClient`

### SV_Frame
- **Signature:** `void SV_Frame(int msec)`
- **Purpose:** Master per-frame entry point: accumulates time residual, runs bot logic, drives `VM_Call(GAME_RUN_FRAME)` in fixed-step chunks, updates configstrings, checks timeouts, sends client snapshots, and sends heartbeats.
- **Inputs:** `msec` — elapsed milliseconds since last call
- **Side effects:** Advances `svs.time` and `sv.timeResidual`; may call `SV_Shutdown` on kill-switch or overflow conditions; calls `Cbuf_AddText` for map restart
- **Calls:** `SV_CheckPaused`, `SV_BotFrame`, `NET_Sleep`, `VM_Call`, `SV_CalcPings`, `SV_CheckTimeouts`, `SV_SendClientMessages`, `SV_MasterHeartbeat`, `SV_Shutdown`, `Cbuf_AddText`, `SV_SetConfigstring`
- **Notes:** 32-bit time wrap (~23 days) and `nextSnapshotEntities` near-overflow both trigger a clean shutdown and map restart.

## Control Flow Notes
`SV_Frame` is called once per engine frame from `Com_Frame` (in `qcommon/common.c`). It is the server's equivalent of the render loop tick. `SV_PacketEvent` is called from the network layer (also in `Com_Frame`) before `SV_Frame`, so client input is already processed when the game VM runs. Init/shutdown are handled in `sv_init.c`; this file owns only the steady-state run loop and packet dispatch.

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `SV_DropClient`, `SV_GetChallenge`, `SV_DirectConnect`, `SV_AuthorizeIpPacket`, `SV_ExecuteClientMessage`, `SV_Netchan_Process`, `SV_BotFrame`, `SV_SendClientMessages`, `SV_SetConfigstring`, `SV_GameClientNum`, `VM_Call`, `NET_OutOfBandPrint`, `NET_StringToAdr`, `NET_Sleep`, `Huff_Decompress`, `Com_BeginRedirect`, `Com_EndRedirect`, `Cbuf_AddText`, `Cvar_InfoString`, `Cvar_InfoString_Big`, `cvar_modifiedFlags`, `com_dedicated`, `com_sv_running`, `cl_paused`, `sv_paused`, `com_speeds`, `time_game`
