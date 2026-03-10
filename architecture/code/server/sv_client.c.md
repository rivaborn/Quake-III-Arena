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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `client_t` | struct (defined in server.h) | Per-client state: net channel, download state, snapshot timing, commands |
| `challenge_t` | struct (defined in server.h) | Tracks pending challenge/response handshakes to prevent DoS |
| `ucmd_t` | struct | Name→function mapping for client-issued reliable commands |
| `clientState_t` | enum (defined in server.h) | `CS_FREE/ZOMBIE/CONNECTED/PRIMED/ACTIVE` lifecycle states |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `ucmds[]` | `static ucmd_t[]` | file-static | Dispatch table for reliable client commands (userinfo, disconnect, cp, download, etc.) |

## Key Functions / Methods

### SV_GetChallenge
- **Signature:** `void SV_GetChallenge( netadr_t from )`
- **Purpose:** Issues a challenge number to a connecting client; contacts the ID authorize server for non-LAN clients.
- **Inputs:** `from` — client network address
- **Outputs/Return:** void; sends OOB `challengeResponse` or `getIpAuthorize` packets
- **Side effects:** Writes to `svs.challenges[]`; sends UDP packets via `NET_OutOfBandPrint`
- **Calls:** `Cvar_VariableValue`, `NET_CompareAdr`, `Sys_IsLANAddress`, `NET_StringToAdr`, `NET_OutOfBandPrint`, `Cvar_Get`
- **Notes:** Bypasses authorization after `AUTHORIZE_TIMEOUT` (5000 ms) if authorize server is unreachable. Single-player mode is silently ignored.

### SV_AuthorizeIpPacket
- **Signature:** `void SV_AuthorizeIpPacket( netadr_t from )`
- **Purpose:** Processes the authorize server's response; sends `challengeResponse` on accept or prints rejection message to client.
- **Inputs:** `from` — must match `svs.authorizeAddress`
- **Outputs/Return:** void; sends OOB packets to waiting client
- **Side effects:** Clears `svs.challenges[i]` on rejection/unknown; calls `NET_OutOfBandPrint`
- **Calls:** `NET_CompareBaseAdr`, `Cmd_Argv`, `NET_OutOfBandPrint`, `Com_Memset`
- **Notes:** Handles `demo`, `accept`, `unknown`, and implicit-deny cases.

### SV_DirectConnect
- **Signature:** `void SV_DirectConnect( netadr_t from )`
- **Purpose:** Processes an incoming `connect` OOB packet: validates protocol/challenge, allocates a `client_t` slot, invokes `GAME_CLIENT_CONNECT`, and transitions to `CS_CONNECTED`.
- **Inputs:** `from` — client address; userinfo string parsed from `Cmd_Argv(1)`
- **Outputs/Return:** void; sends `connectResponse` or rejection print OOB
- **Side effects:** Modifies `svs.clients[]`, initializes netchan, calls VM, may call `SV_Heartbeat_f`
- **Calls:** `VM_Call(GAME_CLIENT_CONNECT)`, `Netchan_Setup`, `SV_UserinfoChanged`, `SV_DropClient`, `SV_Heartbeat_f`, `NET_OutOfBandPrint`
- **Notes:** Private slot reservation via `sv_privateClients`/`sv_privatePassword`. Bot slots can be displaced for local connects.

### SV_DropClient
- **Signature:** `void SV_DropClient( client_t *drop, const char *reason )`
- **Purpose:** Fully disconnects a client, notifies the game VM, clears downloads, and marks the slot `CS_ZOMBIE`.
- **Inputs:** `drop` — client to disconnect; `reason` — human-readable string
- **Side effects:** Sends `disconnect` server command, calls `VM_Call(GAME_CLIENT_DISCONNECT)`, frees bot slot, clears userinfo, may call `SV_Heartbeat_f`
- **Notes:** No-op if already `CS_ZOMBIE`. Does not handle server shutdown (that is `SV_FinalMessage`).

### SV_SendClientGameState
- **Signature:** `void SV_SendClientGameState( client_t *client )`
- **Purpose:** Serializes the full gamestate (configstrings + entity baselines) into a message and delivers it; transitions client to `CS_PRIMED`.
- **Side effects:** Sets `client->gamestateMessageNum`; writes to the message buffer; calls `SV_SendMessageToClient`
- **Calls:** `MSG_Init`, `MSG_WriteLong/Byte/Short/BigString`, `MSG_WriteDeltaEntity`, `SV_UpdateServerCommandsToClient`, `SV_SendMessageToClient`

### SV_ExecuteClientMessage
- **Signature:** `void SV_ExecuteClientMessage( client_t *cl, msg_t *msg )`
- **Purpose:** Top-level parser for every inbound client packet: reads server/message/reliable ACKs, dispatches client commands, then processes movement.
- **Side effects:** Updates `cl->messageAcknowledge`, `cl->reliableAcknowledge`; may resend gamestate or drop client
- **Calls:** `MSG_ReadLong/Byte`, `SV_ClientCommand`, `SV_UserMove`, `SV_SendClientGameState`
- **Notes:** Silently drops malformed ACK values in release builds; drops client in `NDEBUG` builds.

### SV_WriteDownloadToClient
- **Signature:** `void SV_WriteDownloadToClient( client_t *cl, msg_t *msg )`
- **Purpose:** Implements a sliding-window reliable file download protocol; opens the file on first call, reads blocks, and writes `svc_download` records into the snapshot message.
- **Side effects:** Allocates/frees `cl->downloadBlocks[]` via `Z_Malloc`/`Z_Free`; reads from filesystem
- **Notes:** Blocks per snapshot is rate-limited by `cl->rate` and `cl->snapshotMsec`. ID and mission-pack pk3 files are explicitly refused.

### SV_VerifyPaks_f
- **Signature:** `static void SV_VerifyPaks_f( client_t *cl )`
- **Purpose:** On pure servers, validates that the client has loaded exactly the expected cgame/ui checksums and no unauthorized pk3 files.
- **Side effects:** Sets `cl->pureAuthentic` and `cl->gotCP`; may call `SV_DropClient`
- **Notes:** Uses a `while(bGood)` loop with `break` as structured goto. Ignores outdated `cp` sequences from stale gamestate.

### SV_UserMove
- **Signature:** `static void SV_UserMove( client_t *cl, msg_t *msg, qboolean delta )`
- **Purpose:** Reads and decompresses a batch of `usercmd_t` structs, transitions `CS_PRIMED` clients into the world, then feeds each command to `SV_ClientThink`.
- **Calls:** `MSG_ReadDeltaUsercmdKey`, `SV_ClientEnterWorld`, `SV_ClientThink`, `SV_DropClient`, `SV_SendClientGameState`
- **Notes:** Key used for delta decompression is derived from `checksumFeed ^ messageAcknowledge ^ reliableCommands hash` to prevent tampering.

## Control Flow Notes
This file is driven entirely by the **network receive path**:
- `SV_GetChallenge` / `SV_AuthorizeIpPacket` / `SV_DirectConnect` are called from `sv_main.c`'s OOB packet dispatcher during any frame.
- `SV_ExecuteClientMessage` is called per connected client each server frame from `sv_main.c:SV_PacketEvent`.
- `SV_WriteDownloadToClient` is called from `sv_snapshot.c` during snapshot generation for downloading clients.
- No direct render or physics involvement; purely protocol and VM dispatch.

## External Dependencies
- `server.h` → pulls in `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:** `svs` (`serverStatic_t`), `sv` (`server_t`), `gvm` (`vm_t*`), all `sv_*` cvars, `VM_Call`, `Netchan_Setup`, `NET_OutOfBandPrint`, `FS_SV_FOpenFileRead`, `FS_Read`, `FS_idPak`, `FS_LoadedPakPureChecksums`, `MSG_*` family, `SV_Heartbeat_f`, `SV_SendClientSnapshot`, `SV_BotFreeClient`, `SV_GentityNum`
