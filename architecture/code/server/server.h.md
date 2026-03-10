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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `svEntity_t` | struct | Server-side entity metadata: BSP cluster membership, portal area numbers, snapshot dedup counter, baseline for delta compression |
| `serverState_t` | enum | Lifecycle state of the server: `SS_DEAD`, `SS_LOADING`, `SS_GAME` |
| `server_t` | struct | Per-map server state: configstrings, entity array, game VM pointers, timing, model handles; cleared on each map load |
| `serverStatic_t` | struct | Persistent server state across map changes: client array, snapshot entity pool, challenge table, heartbeat timer |
| `clientSnapshot_t` | struct | One recorded snapshot for a client: player state, entity list window, transmission timestamps, area visibility bits |
| `clientState_t` | enum | Client connection lifecycle: `CS_FREE`, `CS_ZOMBIE`, `CS_CONNECTED`, `CS_PRIMED`, `CS_ACTIVE` |
| `netchan_buffer_t` | struct | Linked-list node for queuing outgoing fragmented netchan messages per client |
| `client_t` | struct | Full per-client state: connection state, reliable command ring buffer, download state, snapshot history (`PACKET_BACKUP` frames), rate/ping tracking, netchan |
| `challenge_t` | struct | Connection challenge record: address, token, timestamps, used to prevent spoofed connects |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `svs` | `serverStatic_t` | global | Persistent server data; survives map changes |
| `sv` | `server_t` | global | Per-map server data; cleared each `SV_SpawnServer` |
| `gvm` | `vm_t *` | global | Handle to the game logic virtual machine |
| `sv_fps`, `sv_timeout`, `sv_maxclients`, etc. | `cvar_t *` | global | ~25 server configuration cvars |

## Key Functions / Methods

### SV_SpawnServer
- Signature: `void SV_SpawnServer(char *server, qboolean killBots)`
- Purpose: Load a new map, initialize `sv`, spawn game VM, send gamestate to connected clients.
- Inputs: Map name string, flag to remove bots.
- Outputs/Return: void
- Side effects: Clears `sv`, resets configstrings, restarts game VM, modifies `svs`.
- Calls: Defined in `sv_init.c`; calls into `SV_InitGameProgs`, `SV_SetConfigstring`, collision model loading.
- Notes: Central map-transition entry point.

### SV_Frame (declared in qcommon.h)
- Not declared here but drives calls into `SV_SendClientMessages`, `SV_BotFrame`, game VM `GAME_RUN_FRAME`.

### SV_SendClientMessages
- Signature: `void SV_SendClientMessages(void)`
- Purpose: Per-frame dispatch — build and transmit snapshots to all active clients.
- Side effects: Calls `SV_SendClientSnapshot` per client; writes to netchan.
- Calls: `sv_snapshot.c` functions.

### SV_ExecuteClientMessage
- Signature: `void SV_ExecuteClientMessage(client_t *cl, msg_t *msg)`
- Purpose: Parse an incoming client packet: process user commands and reliable client commands.
- Side effects: Advances `cl->lastUsercmd`, may call `SV_ClientThink`, `SV_ExecuteClientCommand`.

### SV_DropClient
- Signature: `void SV_DropClient(client_t *drop, const char *reason)`
- Purpose: Disconnect a client, transition to `CS_ZOMBIE`, notify game VM.
- Side effects: Sends disconnect message, calls `GAME_CLIENT_DISCONNECT` on VM.

### SV_LinkEntity / SV_UnlinkEntity
- Signature: `void SV_LinkEntity(sharedEntity_t *ent)` / `void SV_UnlinkEntity(sharedEntity_t *ent)`
- Purpose: Insert/remove entity from the BSP world sector tree; updates cluster/area membership in `svEntity_t`.
- Side effects: Modifies `sv.svEntities`, world sector linked lists.
- Notes: Must be called whenever origin, mins, maxs, or solid changes.

### SV_Trace
- Signature: `void SV_Trace(trace_t *results, const vec3_t start, vec3_t mins, vec3_t maxs, const vec3_t end, int passEntityNum, int contentmask, int capsule)`
- Purpose: Full world + entity sweep trace. Combines BSP clip model trace with per-entity checks.
- Outputs/Return: Fills `results` trace_t.
- Notes: `passEntityNum` is excluded from testing (self-exclusion).

### SV_InitGameProgs / SV_ShutdownGameProgs
- Purpose: Load/unload the game VM (`gvm`), register game syscall trap table.
- Side effects: Allocates/frees VM; calls `GAME_INIT`/`GAME_SHUTDOWN`.

### SV_BotFrame
- Signature: `void SV_BotFrame(int time)`
- Purpose: Drive bot AI tick via the bot library for the current server time.
- Side effects: Calls into botlib, may generate usercmds for bot clients.

### SV_Netchan_Transmit / SV_Netchan_Process
- Purpose: Server-side reliable channel send/receive with outgoing fragment queue support (`netchan_buffer_t`).
- Notes: Queues large fragmented messages to avoid UDP burst; defined in `sv_net_chan.c`.

## Control Flow Notes
- `SV_Frame` (called from `Com_Frame` each engine tick) drives: world simulation via game VM → `SV_SendClientMessages` → snapshot build/transmit.
- Init path: `SV_Init` → `SV_SpawnServer` → `SV_InitGameProgs` → `GAME_INIT`.
- Packet path: `SV_PacketEvent` → `SV_ExecuteClientMessage` → `SV_ClientThink` / `SV_ExecuteClientCommand`.
- Shutdown: `SV_Shutdown` → `SV_ShutdownGameProgs` → client disconnects.

## External Dependencies
- `../game/q_shared.h` — shared types: `vec3_t`, `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t`, `cvar_t`, `netadr_t`, etc.
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `vm_t`, `PACKET_BACKUP`, `MAX_MSGLEN`, filesystem, cvar, cmd APIs
- `../game/g_public.h` — `sharedEntity_t`, `entityShared_t`, `gameImport_t`/`gameExport_t` trap enums, `SVF_*` flags
- `../game/bg_public.h` — `pmove_t`, game constants, configstring index definitions
- **Defined elsewhere:** `worldSector_s` (sv_world.c), `cmodel_s` (collision model system), all `SV_*` function bodies across `sv_*.c` files, `vm_t` (vm.c)
