# code/server/server.h
## File Purpose
Central header for the Quake III Arena dedicated server subsystem. Defines all major server-side data structures, global state variables, and declares the full public API surface across all server `.c` modules (`sv_main`, `sv_init`, `sv_client`, `sv_snapshot`, `sv_game`, `sv_bot`, `sv_world`, `sv_net_chan`).

## Core Responsibilities
- Define the per-frame server state (`server_t`) and persistent cross-map server state (`serverStatic_t`)
- Define per-client state (`client_t`), per-snapshot state (`clientSnapshot_t`), and connection handshake state (`challenge_t`)
- Define the server-side entity wrapper (`svEntity_t`) used for spatial partitioning and PVS/cluster tracking
- Declare all server cvars as extern pointers
- Declare all inter-module function prototypes for the server subsystem
- Expose the spatial world-query API (link/unlink, area queries, traces, point contents)

## External Dependencies
- `../game/q_shared.h` — shared types: `vec3_t`, `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t`, `cvar_t`, `netadr_t`, etc.
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `vm_t`, `PACKET_BACKUP`, `MAX_MSGLEN`, filesystem, cvar, cmd APIs
- `../game/g_public.h` — `sharedEntity_t`, `entityShared_t`, `gameImport_t`/`gameExport_t` trap enums, `SVF_*` flags
- `../game/bg_public.h` — `pmove_t`, game constants, configstring index definitions
- **Defined elsewhere:** `worldSector_s` (sv_world.c), `cmodel_s` (collision model system), all `SV_*` function bodies across `sv_*.c` files, `vm_t` (vm.c)

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

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- `botlib.h` — defines `botlib_import_t`, `botlib_export_t`, `bsp_trace_t`, `bot_input_t`
- **Defined elsewhere:** `SV_Trace`, `SV_ClipToEntity`, `SV_PointContents`, `SV_inPVS`, `SV_ExecuteClientCommand`, `SV_GentityNum`, `CM_EntityString`, `CM_InlineModel`, `CM_ModelBounds`, `RadiusFromBounds`, `Z_TagMalloc`, `Z_Free`, `Z_Malloc`, `Z_AvailableMemory`, `Hunk_Alloc`, `Hunk_CheckMark`, `VM_Call`, `GetBotLibAPI`, `Sys_CheckCD`, `Cvar_Get`, `Cvar_VariableIntegerValue`, `Cvar_VariableValue`, `FS_FOpenFileByMode`, `FS_Read2`, `FS_Write`, `FS_FCloseFile`, `FS_Seek`, `gvm` (game VM handle), `svs`/`sv` server state globals.

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

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `svs`, `sv`, `gvm`, `sv_maxclients`, `sv_gametype`, `sv_mapname`, `com_sv_running`, `com_frameTime`, `com_dedicated`; `SV_SpawnServer`, `SV_DropClient`, `SV_ClientEnterWorld`, `SV_RestartGameProgs`, `SV_AddServerCommand`, `SV_SendServerCommand`, `SV_SetConfigstring`, `SV_GameClientNum`, `SV_SectorList_f`; `VM_Call`, `VM_ExplicitArgPtr`; `NET_StringToAdr`, `NET_OutOfBandPrint`, `NET_AdrToString`; `Cmd_*`, `Cvar_*`, `Info_Print`, `FS_ReadFile`

# code/server/sv_client.c
## File Purpose
Handles all server-side client lifecycle management for Quake III Arena, from initial connection negotiation and authorization through in-game command processing, file downloads, and disconnection. It is the primary interface between raw network messages from clients and the game VM.

## Core Responsibilities
- Challenge/response handshake to prevent spoofed connections (`SV_GetChallenge`, `SV_AuthorizeIpPacket`)
- Direct connection processing: protocol validation, challenge verification, slot allocation (`SV_DirectConnect`)
- Client state transitions: `CS_FREE` → `CS_CONNECTED` → `CS_PRIMED` → `CS_ACTIVE` → `CS_ZOMBIE`
- Gamestate serialization and transmission to newly connected/map-restarted clients (`SV_SendClientGameState`)
- In-game packet parsing: client commands, user movement, flood protection (`SV_ExecuteClientMessage`)
- Pure server pak checksum validation (`SV_VerifyPaks_f`)
- Sliding-window file download streaming (`SV_WriteDownloadToClient`)
- Client disconnection and cleanup (`SV_DropClient`)

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `svs` (`serverStatic_t`), `sv` (`server_t`), `gvm` (`vm_t*`), all `sv_*` cvars, `VM_Call`, `Netchan_Setup`, `NET_OutOfBandPrint`, `FS_SV_FOpenFileRead`, `FS_Read`, `FS_idPak`, `FS_LoadedPakPureChecksums`, `MSG_*` family, `SV_Heartbeat_f`, `SV_SendClientSnapshot`, `SV_BotFreeClient`, `SV_GentityNum`

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

## External Dependencies
- `server.h` — `svs`, `sv`, `gvm`, all server types and function declarations
- `../game/botlib.h` — `botlib_export_t`, all `BOTLIB_*` syscall constants
- **Defined elsewhere:** `VM_Create`, `VM_Call`, `VM_Free`, `VM_Restart`, `VM_ArgPtr`; all `CM_*` collision functions; `SV_LinkEntity`, `SV_UnlinkEntity`, `SV_Trace`, `SV_AreaEntities`; `SV_BotAllocateClient`, `SV_BotLibSetup`, `SV_BotGetSnapshotEntity`; `BotImport_DebugPolygonCreate/Delete`; `FS_*`, `Cvar_*`, `Cmd_*`, `Cbuf_*`, `Com_*`, `Sys_*`, `MatrixMultiply`, `AngleVectors`, `PerpendicularVector`

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

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `Z_Free`, `Z_Malloc`, `CopyString`, `Hunk_Alloc/Clear/SetMark`, `Hunk_AllocateTempMemory`, `VM_Call`, `VM_ExplicitArgPtr`, `FS_Restart`, `FS_ClearPakReferences`, `FS_LoadedPakChecksums/Names`, `FS_ReferencedPakChecksums/Names`, `CM_LoadMap`, `CM_ClearMap`, `CL_MapLoading`, `CL_ShutdownAll`, `CL_Disconnect`, `Cvar_Get/Set/VariableValue/InfoString/InfoString_Big`, `Com_Printf/Error/Milliseconds/Memset`, `SV_InitGameProgs`, `SV_ShutdownGameProgs`, `SV_ClearWorld`, `SV_SendServerCommand`, `SV_SendClientSnapshot`, `SV_DropClient`, `SV_BotFrame`, `SV_BotInitCvars`, `SV_BotInitBotLib`, `SV_Heartbeat_f`, `SV_MasterShutdown`, `SV_AddOperatorCommands`, `SV_RemoveOperatorCommands`

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

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `SV_DropClient`, `SV_GetChallenge`, `SV_DirectConnect`, `SV_AuthorizeIpPacket`, `SV_ExecuteClientMessage`, `SV_Netchan_Process`, `SV_BotFrame`, `SV_SendClientMessages`, `SV_SetConfigstring`, `SV_GameClientNum`, `VM_Call`, `NET_OutOfBandPrint`, `NET_StringToAdr`, `NET_Sleep`, `Huff_Decompress`, `Com_BeginRedirect`, `Com_EndRedirect`, `Cbuf_AddText`, `Cvar_InfoString`, `Cvar_InfoString_Big`, `cvar_modifiedFlags`, `com_dedicated`, `com_sv_running`, `cl_paused`, `sv_paused`, `com_speeds`, `time_game`

# code/server/sv_net_chan.c
## File Purpose
Provides server-side network channel wrapper functions that layer XOR-based obfuscation encoding/decoding on top of the base `Netchan` fragmentation and sequencing layer. It also manages a per-client outgoing message queue to prevent UDP packet bursts when large fragmented messages collide during transmission.

## Core Responsibilities
- XOR-encode outgoing server messages using client challenge, sequence number, and acknowledged command strings as a rolling key
- XOR-decode incoming client messages using matching key material
- Queue outgoing messages when the netchan already has unsent fragments, ensuring correct ordering
- Drain the outgoing queue by encoding and transmitting the next queued message once fragmentation completes
- Wrap `Netchan_Process` with a decode step for all received client packets

## External Dependencies
- `../game/q_shared.h` — base types (`byte`, `qboolean`, `msg_t` primitives)
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `Netchan_Transmit`, `Netchan_TransmitNextFragment`, `Netchan_Process`, `MSG_ReadLong`, `MSG_WriteByte`, `MSG_Copy`, `Z_Malloc`, `Z_Free`, `Com_DPrintf`, `Com_Error`; constants `SV_ENCODE_START`, `SV_DECODE_START`, `MAX_RELIABLE_COMMANDS`, `svc_EOF`
- `server.h` — `client_t`, `netchan_buffer_t`, `MAX_MSGLEN`
- `Netchan_*` functions — defined in `qcommon/net_chan.c` (not this file)

# code/server/sv_rankings.c
## File Purpose
Implements the server-side interface to Id Software's Global Rankings (GRank) system, managing player authentication, match tracking, and stat reporting via an external rankings API. It bridges Quake III Arena's server loop with the asynchronous GRank library using callback-based operations.

## Core Responsibilities
- Initialize and shut down the GRank rankings session per game match
- Authenticate players via server-side login/create or client-side token validation
- Track per-player GRank contexts, match handles, and player IDs
- Submit integer and string stat reports for players/server during gameplay
- Handle asynchronous GRank callbacks for new game, login, join game, send reports, and cleanup
- Encode/decode player IDs and tokens using a custom 6-bit ASCII encoding scheme
- Manage context reference counting to safely free resources when all contexts close

## External Dependencies
- `server.h` — server types, cvars (`sv_maxclients`, `sv_enableRankings`, `sv_rankingsActive`), `SV_SetConfigstring`, `Z_Malloc`, `Z_Free`, `Cvar_Set`, `Cvar_VariableValue`, `Com_DPrintf`
- `../rankings/1.0/gr/grapi.h` — GRank API: `GRankInit`, `GRankNewGameAsync`, `GRankUserLoginAsync`, `GRankUserCreateAsync`, `GRankJoinGameAsync`, `GRankPlayerValidate`, `GRankSendReportsAsync`, `GRankCleanupAsync`, `GRankStartMatch`, `GRankReportInt`, `GRankReportStr`, `GRankPoll`; types `GR_CONTEXT`, `GR_STATUS`, `GR_PLAYER_TOKEN`, `GR_NEWGAME`, `GR_LOGIN`, `GR_JOINGAME`, `GR_MATCH`, `GR_INIT` — **defined in external rankings library, not in this file**
- `../rankings/1.0/gr/grlog.h` — `GRankLogLevel`, `GRLOG_OFF`, `GRLOG_TRACE` — **defined in external rankings library**
- `LittleLong64` — byte-order conversion for 64-bit values — **defined elsewhere in qcommon**

# code/server/sv_snapshot.c
## File Purpose
Builds per-client game snapshots each server frame and transmits them over the network using delta compression. It determines entity visibility via PVS/area checks, encodes state deltas, and throttles transmission via rate control.

## Core Responsibilities
- Build `clientSnapshot_t` frames by culling visible entities per PVS and area connectivity
- Delta-encode entity states (`entityState_t` list) between frames for bandwidth efficiency
- Delta-encode `playerState_t` between frames
- Write the full snapshot packet (header, areabits, playerstate, entities) to a `msg_t`
- Retransmit unacknowledged reliable server commands to clients
- Throttle snapshot delivery using per-client rate and `snapshotMsec` limits
- Drive the per-frame send loop across all connected clients

## External Dependencies
- **Includes:** `server.h` → `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:**
  - `MSG_WriteDeltaEntity`, `MSG_WriteDeltaPlayerstate`, `MSG_WriteByte/Long/Bits/Data/String`, `MSG_Init`, `MSG_Clear` — `qcommon/msg.c`
  - `CM_PointLeafnum`, `CM_LeafArea`, `CM_LeafCluster`, `CM_ClusterPVS`, `CM_AreasConnected`, `CM_WriteAreaBits` — `qcommon/cm_*.c`
  - `SV_Netchan_Transmit`, `SV_Netchan_TransmitNextFragment` — `sv_net_chan.c`
  - `SV_GentityNum`, `SV_GameClientNum`, `SV_SvEntityForGentity` — `sv_game.c`
  - `SV_WriteDownloadToClient` — `sv_client.c`
  - `svs`, `sv`, `sv_padPackets`, `sv_maxRate`, `sv_lanForceRate`, `sv_maxclients` — globals/cvars

# code/server/sv_world.c
## File Purpose
Implements server-side spatial partitioning and world query operations for Quake III Arena. It maintains an axis-aligned BSP sector tree for fast entity lookups and provides collision tracing, area queries, and point-contents testing against both world geometry and game entities.

## Core Responsibilities
- Build and manage a uniform spatial subdivision tree (`worldSector_t`) for entity bucketing
- Link/unlink game entities into the sector tree when they move or change bounds
- Compute and cache PVS cluster memberships and area numbers per entity on link
- Query all entities whose AABBs overlap a given region (`SV_AreaEntities`)
- Perform swept-box traces through the world and all solid entities (`SV_Trace`)
- Clip a movement against a single specific entity (`SV_ClipToEntity`)
- Return combined content flags at a world point across all overlapping entities (`SV_PointContents`)

## External Dependencies
- **`server.h`** → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `CM_InlineModel`, `CM_ModelBounds`, `CM_BoxLeafnums`, `CM_LeafArea`, `CM_LeafCluster`, `CM_BoxTrace`, `CM_TransformedBoxTrace`, `CM_TransformedPointContents`, `CM_PointContents`, `CM_TempBoxModel`; `SV_SvEntityForGentity`, `SV_GEntityForSvEntity`, `SV_GentityNum`; `RadiusFromBounds`; globals `sv`, `svs`

