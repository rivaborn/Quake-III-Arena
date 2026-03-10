# code/client/cl_parse.c

## File Purpose
Parses incoming server-to-client network messages for Quake III Arena. It decodes the server message stream into snapshots, entity states, game state, downloads, and server commands that the client uses to update its local world representation.

## Core Responsibilities
- Dispatch incoming server messages by opcode (`svc_*`)
- Parse full game state on level load/connection (configstrings + entity baselines)
- Parse delta-compressed snapshots (player state + packet entities)
- Reconstruct entity states via delta decompression from prior frames or baselines
- Handle file download protocol (block-based chunked transfer)
- Store server command strings for deferred cgame execution
- Sync client-side cvars from server `systeminfo` configstring

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `clSnapshot_t` | struct (defined in client.h) | One frame of server state: player state, entity list, flags, delta info |
| `clientActive_t` (`cl`) | struct / global | Ring buffers for snapshots, parse entities, baselines, outgoing packets |
| `clientConnection_t` (`clc`) | struct / global | Connection state: sequences, download state, demo flags, netchan |
| `msg_t` | struct (qcommon) | Bitstream read/write buffer passed through all parse functions |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `svc_strings` | `char*[256]` | file-static (global linkage) | Human-readable names for `svc_*` opcodes; used by `SHOWNET` debug output |
| `cl_connectedToPureServer` | `int` | global | Tracks whether the server is running in pure mode; exported to other client files |

## Key Functions / Methods

### SHOWNET
- **Signature:** `void SHOWNET(msg_t *msg, char *s)`
- **Purpose:** Debug helper that prints message offset and label when `cl_shownet >= 2`
- **Inputs:** Current message buffer, label string
- **Outputs/Return:** None
- **Side effects:** `Com_Printf` to console
- **Calls:** `Com_Printf`

---

### CL_DeltaEntity
- **Signature:** `void CL_DeltaEntity(msg_t *msg, clSnapshot_t *frame, int newnum, entityState_t *old, qboolean unchanged)`
- **Purpose:** Decodes one entity into `cl.parseEntities[]` ring buffer, either copying unchanged or applying a delta. Increments `frame->numEntities` and `cl.parseEntitiesNum`.
- **Inputs:** Message stream, destination frame, entity number, old state to delta from, flag for unchanged copy
- **Outputs/Return:** None (writes into `cl.parseEntities` ring)
- **Side effects:** Modifies `cl.parseEntitiesNum`, `frame->numEntities`
- **Calls:** `MSG_ReadDeltaEntity`
- **Notes:** Entity number `MAX_GENTITIES-1` is a sentinel meaning "delta removed"; function returns early without counting it.

---

### CL_ParsePacketEntities
- **Signature:** `void CL_ParsePacketEntities(msg_t *msg, clSnapshot_t *oldframe, clSnapshot_t *newframe)`
- **Purpose:** Merges old-frame entities with incoming delta stream into `newframe`. Handles three cases per entity: unchanged carry-forward, delta from prior state, delta from baseline.
- **Inputs:** Message stream, previous snapshot (may be NULL), new snapshot being built
- **Outputs/Return:** None (populates `newframe`)
- **Side effects:** Indirectly modifies `cl.parseEntities[]` via `CL_DeltaEntity`
- **Calls:** `MSG_ReadBits`, `CL_DeltaEntity`, `Com_Printf`, `Com_Error`
- **Notes:** Walks two sorted lists (old entities by number, incoming stream by number) in a merge-scan pattern; any remaining old entities are carried forward at the end.

---

### CL_ParseSnapshot
- **Signature:** `void CL_ParseSnapshot(msg_t *msg)`
- **Purpose:** Reads a full snapshot from the message, validates delta availability, and if valid commits it to `cl.snap` and `cl.snapshots[]`. Computes ping from outPacket timestamps.
- **Inputs:** Message stream
- **Outputs/Return:** None
- **Side effects:** Writes `cl.snap`, `cl.snapshots[]`, `cl.newSnapshots`, invalidates stale snapshot slots
- **Calls:** `MSG_ReadLong`, `MSG_ReadByte`, `MSG_ReadData`, `MSG_ReadDeltaPlayerstate`, `CL_ParsePacketEntities`, `Com_Memset`, `Com_Printf`
- **Notes:** If delta base is missing or too old, the snapshot is parsed (to advance read cursor) but discarded. Ping is computed by scanning `cl.outPackets[]` for matching `commandTime`.

---

### CL_SystemInfoChanged
- **Signature:** `void CL_SystemInfoChanged(void)`
- **Purpose:** Re-parses the `CS_SYSTEMINFO` configstring, updates `cl.serverId`, propagates all key/value pairs as cvars, and syncs pure-server pak lists.
- **Inputs:** None (reads from `cl.gameState`)
- **Outputs/Return:** None
- **Side effects:** Multiple `Cvar_Set` calls, `FS_PureServerSetLoadedPaks`, `FS_PureServerSetReferencedPaks`, `Cvar_SetCheatState`
- **Calls:** `Info_ValueForKey`, `atoi`, `Cvar_SetCheatState`, `FS_PureServerSetLoadedPaks`, `FS_PureServerSetReferencedPaks`, `Info_NextPair`, `Cvar_Set`, `Cvar_VariableString`, `Cvar_VariableValue`
- **Notes:** Skipped entirely during demo playback.

---

### CL_ParseGamestate
- **Signature:** `void CL_ParseGamestate(msg_t *msg)`
- **Purpose:** Handles `svc_gamestate`: resets client state, reads all configstrings and entity baselines, then triggers download/cgame init.
- **Inputs:** Message stream
- **Outputs/Return:** None
- **Side effects:** Calls `CL_ClearState`, populates `cl.gameState`, `cl.entityBaselines[]`, calls `CL_SystemInfoChanged`, `FS_ConditionalRestart`, `CL_InitDownloads`, sets `cl_paused`
- **Calls:** `Con_Close`, `CL_ClearState`, `MSG_ReadLong`, `MSG_ReadByte`, `MSG_ReadShort`, `MSG_ReadBigString`, `MSG_ReadBits`, `MSG_ReadDeltaEntity`, `CL_SystemInfoChanged`, `FS_ConditionalRestart`, `CL_InitDownloads`, `Cvar_Set`, `Com_Memcpy`, `Com_Error`

---

### CL_ParseDownload
- **Signature:** `void CL_ParseDownload(msg_t *msg)`
- **Purpose:** Receives one block of a file being downloaded from the server, writes it to a temp file, acknowledges with `nextdl`, and finalizes on zero-length block.
- **Inputs:** Message stream
- **Outputs/Return:** None
- **Side effects:** File I/O (`FS_SV_FOpenFileWrite`, `FS_Write`, `FS_FCloseFile`, `FS_SV_Rename`), `CL_AddReliableCommand`, `CL_WritePacket`, `CL_NextDownload`, cvar updates
- **Notes:** Block 0 is special — carries total file size. Sequence mismatch causes silent discard without disconnecting.

---

### CL_ParseCommandString
- **Signature:** `void CL_ParseCommandString(msg_t *msg)`
- **Purpose:** Stores a reliable server command string into `clc.serverCommands[]` ring if not already seen.
- **Inputs:** Message stream
- **Outputs/Return:** None
- **Side effects:** Writes `clc.serverCommandSequence`, `clc.serverCommands[]`
- **Calls:** `MSG_ReadLong`, `MSG_ReadString`, `Q_strncpyz`

---

### CL_ParseServerMessage
- **Signature:** `void CL_ParseServerMessage(msg_t *msg)`
- **Purpose:** Top-level dispatcher. Reads reliable ACK, then loops reading opcodes and routing to the appropriate sub-parser.
- **Inputs:** Message stream
- **Outputs/Return:** None
- **Side effects:** Updates `clc.reliableAcknowledge`; side effects of all sub-parsers
- **Calls:** `MSG_Bitstream`, `MSG_ReadLong`, `MSG_ReadByte`, `SHOWNET`, `CL_ParseCommandString`, `CL_ParseGamestate`, `CL_ParseSnapshot`, `CL_ParseDownload`, `Com_Printf`, `Com_Error`

## Control Flow Notes
Called each frame from `CL_ReadPackets` → `CL_ParseServerMessage`. The main loop runs inside `CL_ParseServerMessage` dispatching opcodes until `svc_EOF`. `CL_ParseGamestate` is a level-load event that resets all client state and triggers the download/cgame pipeline. `CL_ParseSnapshot` runs every frame during gameplay and drives `cl.newSnapshots`, which `CL_SetCGameTime` watches to advance the cgame.

## External Dependencies
- **Includes:** `client.h` (pulls in `q_shared.h`, `qcommon.h`, `tr_public.h`, `ui_public.h`, `keys.h`, `snd_public.h`, `cg_public.h`, `bg_public.h`)
- **Defined elsewhere:** `cl` (`clientActive_t`), `clc` (`clientConnection_t`), `cls` (`clientStatic_t`), `cl_shownet` (cvar), `MSG_Read*` family (msg.c), `FS_*` (files.c), `Cvar_*` (cvar.c), `CL_AddReliableCommand` / `CL_WritePacket` / `CL_NextDownload` / `CL_ClearState` / `CL_InitDownloads` (cl_main.c), `Con_Close` (console), `Info_*` (q_shared.c)
