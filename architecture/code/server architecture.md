# Subsystem Overview

## Purpose
The server subsystem drives the authoritative Quake III Arena game simulation: it owns the server frame loop, manages all connected client lifecycles, hosts the game VM, and routes all UDP network traffic. It bridges raw network I/O, the collision/BSP world, and the game VM through a tightly coupled set of modules sharing a common header (`server.h`).

## Key Files

| File | Role |
|---|---|
| `code/server/server.h` | Central header; defines `server_t`, `serverStatic_t`, `client_t`, `clientSnapshot_t`, `svEntity_t`, all cvar externs, and all inter-module function prototypes |
| `code/server/sv_main.c` | Frame driver and UDP dispatch hub; owns `svs`, `sv`, `gvm` globals; routes connectionless and sequenced packets; runs the per-frame game tick |
| `code/server/sv_init.c` | Server and map initialization/shutdown; manages configstrings, client array allocation, BSP loading, game VM init, and baseline creation |
| `code/server/sv_client.c` | Client lifecycle: challenge/connect handshake, gamestate serialization, in-game message parsing, file downloads, disconnection |
| `code/server/sv_game.c` | Game VM interface; dispatches all VM→engine system calls via `SV_GameSystemCalls`; manages VM load/restart/shutdown |
| `code/server/sv_snapshot.c` | Per-client snapshot builder; PVS/area culling, delta-encoded entity and playerstate transmission, rate throttling |
| `code/server/sv_world.c` | Spatial partitioning (sector tree); entity link/unlink, area queries, swept-box traces, point-contents tests |
| `code/server/sv_bot.c` | Bot bridge; implements `botlib_import_t` vtable, allocates bot client slots, drives bot AI frame tick |
| `code/server/sv_net_chan.c` | Network channel wrapper; XOR obfuscation encode/decode over `Netchan`, outgoing message queue to prevent burst collisions |
| `code/server/sv_ccmds.c` | Operator/admin console commands: map load, kick/ban, status reporting, server lifecycle control |
| `code/server/sv_rankings.c` | Optional GRank integration; player authentication, match tracking, async stat reporting to external rankings API |

## Core Responsibilities

- **Frame simulation**: Run the authoritative game simulation tick each frame via `VM_Call(gvm, GAME_RUN_FRAME, ...)`, preceded by timeout detection and followed by snapshot dispatch to all connected clients.
- **Client state machine**: Advance each client through `CS_FREE → CS_CONNECTED → CS_PRIMED → CS_ACTIVE → CS_ZOMBIE`, serializing and transmitting gamestate on transitions and cleaning up on disconnect or timeout.
- **Game VM hosting**: Load the game module as a `vm_t` (native DLL or QVM bytecode); serve all VM→engine system calls (collision, entity queries, configstrings, filesystem, cvar, bot calls) through a single dispatch function.
- **Snapshot delivery**: Each frame, build a per-client `clientSnapshot_t` by culling entities via PVS/area-connectivity, delta-encode entity and playerstate deltas against the client's last acknowledged snapshot, and transmit with rate control.
- **Spatial world management**: Maintain a sector-tree over all linked game entities; answer area-entity, swept-trace, and point-contents queries used by both the game VM and the bot library.
- **Network dispatch**: Accept all inbound UDP packets; route connectionless packets (status, info, challenge, connect, rcon) and sequenced in-game packets to the appropriate client handler.
- **Bot AI integration**: Expose engine services (trace, PVS, memory, file I/O) to the BotLib through the `botlib_import_t` vtable; drive per-frame bot AI ticks and manage bot pseudo-client slots.
- **Operator administration**: Expose server management commands (map loading, kick/ban, status, rcon) restricted to the local console or authenticated remote operators.

## Key Interfaces & Data Flow

**Exposed to other subsystems:**
- `SV_Frame(int msec)` — called by the common frame loop (`qcommon`) to advance the server one frame.
- `SV_PacketEvent(netadr_t from, msg_t *msg)` — called by the network layer to deliver an inbound UDP packet.
- `SV_Init()` / `SV_Shutdown()` — called by the common layer at engine startup/shutdown.
- `SV_SpawnServer(char *server)` — called by `sv_ccmds.c` (and transitively by the client for local listen servers) to load a new map.
- `SV_LinkEntity` / `SV_UnlinkEntity`, `SV_Trace`, `SV_AreaEntities`, `SV_PointContents` — world query API consumed by the game VM via `SV_GameSystemCalls`.
- `SV_SetConfigstring` / `SV_GetConfigstring` — configstring store exposed to the game VM and operator commands.
- `svs` (`serverStatic_t`) and `sv` (`server_t`) — shared globals read by multiple modules within the subsystem.

**Consumed from other subsystems:**
- **`qcommon`**: `Netchan_*` fragmentation/sequencing, `MSG_*` bit-stream encoding, `CM_*` BSP collision model, `VM_Create/VM_Call/VM_Free`, filesystem (`FS_*`), cvar (`Cvar_*`), command buffer (`Cmd_*`, `Cbuf_*`), memory (`Z_Malloc/Z_Free`, `Hunk_*`), `NET_*` send/receive primitives.
- **`game/botlib`**: `botlib_export_t` vtable obtained via `GetBotLibAPI`; called for all bot AI operations and AAS queries.
- **`game` VM** (`gvm`): Game logic for `GAME_RUN_FRAME`, `GAME_CLIENT_*` events, `BOTAI_START_FRAME`, etc., invoked entirely through `VM_Call`.
- **`rankings` library** (optional): External `grapi.h` / GRank async API for global player authentication and stat reporting.

## Runtime Role

- **Init**: `SV_Init` registers all server cvars and operator commands. `SV_SpawnServer` then clears previous state, calls `CM_LoadMap` to load the BSP, allocates/reallocates the `svs.clients` array, calls `SV_InitGameProgs` to load the game VM, runs several settling frames, builds delta baselines, and transitions any already-connected clients into the new level.
- **Frame** (`SV_Frame`): Checks for client timeouts and zombie cleanup → calls `VM_Call(gvm, GAME_RUN_FRAME)` → calls `SV_BotFrame` for bot AI ticks → calls `SV_SendClientMessages` (which builds and transmits snapshots for each active client via `sv_snapshot.c`) → sends heartbeats to master servers as needed. Incoming packets arrive via `SV_PacketEvent` (dispatched from the common network poll before or interleaved with the frame).
- **Shutdown**: `SV_Shutdown` sends `disconnect` to all connected clients, calls `SV_ShutdownGameProgs` to unload the game VM, notifies master servers, and frees the client array and all server-side memory.

## Notable Implementation Details

- **XOR obfuscation layer** (`sv_net_chan.c`): Outgoing server messages are XOR-keyed using the client's challenge value, the outgoing sequence number, and the last acknowledged reliable command string concatenated as a rolling key. This is a lightweight anti-sniffing measure, not cryptographic security. An outgoing queue (`netchan_buffer_t` linked list) per client serializes large fragmented messages to prevent out-of-order delivery.
- **Sector-tree spatial partitioning** (`sv_world.c`): Uses a static recursive axis-aligned BSP (not the BSP file's own tree) with a fixed depth to bucket entities; `SV_LinkEntity` recomputes PVS cluster and area numbers from the entity's origin at link time, caching them in `svEntity_t` for fast snapshot culling.
- **QVM system call dispatch** (`sv_game.c`): The entire game→engine API surface is a single `intptr_t SV_GameSystemCalls(intptr_t *args)` switch; the game VM (whether native or interpreted QVM bytecode) calls into it via the `vm_t` trap mechanism, keeping the VM sandboxed from direct engine symbol access.
- **Delta snapshot baseline** (`sv_init.c` / `sv_snapshot.c`): After map load, `SV_SpawnServer` runs `NUM_BASELINES` settling frames and records `entityState_t` baselines. Snapshots subsequently delta-encode each entity against these baselines (or the client's last acknowledged frame), minimizing bandwidth for slowly changing or static entities.
- **Client state zombie window** (`sv_client.c`): Disconnected clients linger in `CS_ZOMBIE` state for a configurable interval to absorb duplicate/delayed network packets and prevent slot reuse from confusing in-flight datagrams.
- **Bot pseudo-clients** (`sv_bot.c`): Bots occupy real `client_t` slots and receive snapshots identically to human clients; the distinction is that their `usercmd_t` input is injected by the game VM's bot AI rather than read from the network.
- **Rankings integration** (`sv_rankings.c`): Entirely optional; gated by the `sv_enableRankings` cvar. All GRank operations are asynchronous callbacks polled each server frame via `GRankPoll`. A custom 6-bit ASCII codec encodes 64-bit player tokens for transmission through the configstring system.
