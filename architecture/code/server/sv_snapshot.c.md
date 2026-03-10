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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `snapshotEntityNumbers_t` | struct | Temporary, per-build list of up to 1024 entity numbers visible in one snapshot |
| `clientSnapshot_t` | struct (defined in server.h) | Stores a completed frame: areabits, playerstate, entity range into circular buffer |
| `svEntity_t` | struct (defined in server.h) | Server-side entity tracking: clusters, areas, snapshot counter for dedup |
| `client_t` | struct (defined in server.h) | Full client state including frame ring buffer, rate, netchan |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `svs` | `serverStatic_t` | global (extern) | Persistent server state: client array, snapshot entity circular buffer, time |
| `sv` | `server_t` | global (extern) | Per-map server state: entity array, `snapshotCounter`, `num_entities` |

## Key Functions / Methods

### SV_EmitPacketEntities
- **Signature:** `static void SV_EmitPacketEntities(clientSnapshot_t *from, clientSnapshot_t *to, msg_t *msg)`
- **Purpose:** Writes a delta-compressed entity list by merging the old and new snapshot entity arrays in sorted order.
- **Inputs:** Old frame (`from`, may be NULL), new frame (`to`), output message buffer.
- **Outputs/Return:** Writes directly to `msg`.
- **Side effects:** None beyond `msg` writes.
- **Calls:** `MSG_WriteDeltaEntity`, `MSG_WriteBits`
- **Notes:** Merges two sorted entity lists; sentinel value `9999` handles list exhaustion. Terminates with `MAX_GENTITIES-1` marker. New entities baseline-delta from `sv.svEntities[n].baseline`; removed entities are explicitly removed with `force=qtrue, newent=NULL`.

### SV_WriteSnapshotToClient
- **Signature:** `static void SV_WriteSnapshotToClient(client_t *client, msg_t *msg)`
- **Purpose:** Serializes one complete snapshot packet for a client, choosing an appropriate delta base frame.
- **Inputs:** Client pointer, output `msg_t`.
- **Outputs/Return:** Writes to `msg`.
- **Side effects:** May call `Cvar_Set` (via `SV_RateMsec` indirectly). Logs debug prints for stale delta requests.
- **Calls:** `MSG_WriteByte`, `MSG_WriteLong`, `MSG_WriteData`, `MSG_WriteDeltaPlayerstate`, `SV_EmitPacketEntities`, `Com_DPrintf`
- **Notes:** Falls back to full (non-delta) send if `deltaMessage` is too old or entities rolled off the circular buffer (`first_entity <= svs.nextSnapshotEntities - svs.numSnapshotEntities`). Adds padding NOPs when `sv_padPackets` is set.

### SV_AddEntitiesVisibleFromPoint
- **Signature:** `static void SV_AddEntitiesVisibleFromPoint(vec3_t origin, clientSnapshot_t *frame, snapshotEntityNumbers_t *eNums, qboolean portal)`
- **Purpose:** Iterates all server entities and adds those visible from `origin` via PVS cluster bits and area connectivity.
- **Inputs:** View origin, frame being built, entity number accumulator, portal recursion flag.
- **Outputs/Return:** Populates `eNums` and `frame->areabits`.
- **Side effects:** Writes `frame->areabytes`/`areabits` (OR'd across portal recursions). Recursively calls itself for `SVF_PORTAL` entities.
- **Calls:** `CM_PointLeafnum`, `CM_LeafArea`, `CM_LeafCluster`, `CM_WriteAreaBits`, `CM_ClusterPVS`, `CM_AreasConnected`, `SV_GentityNum`, `SV_SvEntityForGentity`, `SV_AddEntToSnapshot`, `VectorSubtract`, `VectorLengthSquared`, `Com_DPrintf`, `Com_Error`
- **Notes:** Honors `SVF_NOCLIENT`, `SVF_SINGLECLIENT`, `SVF_NOTSINGLECLIENT`, `SVF_CLIENTMASK`, `SVF_BROADCAST`. Uses `svEnt->snapshotCounter` to prevent double-adding through portals. `SVF_CLIENTMASK` is limited to clients 0–31.

### SV_BuildClientSnapshot
- **Signature:** `static void SV_BuildClientSnapshot(client_t *client)`
- **Purpose:** Constructs a full `clientSnapshot_t` for one client: copies playerstate, gathers visible entities, sorts them, inverts areabits, and stores entity states into the global circular buffer.
- **Inputs:** Client to snapshot.
- **Outputs/Return:** Populates `client->frames[outgoingSequence & PACKET_MASK]` and advances `svs.nextSnapshotEntities`.
- **Side effects:** Increments `sv.snapshotCounter`. Advances `svs.nextSnapshotEntities`. Writes to `svs.snapshotEntities[]` ring buffer. Calls `Com_Error(ERR_FATAL)` if counter wraps.
- **Calls:** `SV_GameClientNum`, `SV_GentityNum`, `SV_AddEntitiesVisibleFromPoint`, `qsort`/`SV_QsortEntityNumbers`, `Com_Memset`, `Com_Error`, `VectorCopy`
- **Notes:** Excludes the client's own entity (set `snapshotCounter` to prevent it being added). Areabits are XOR-inverted after all portals are processed to convert visible→mask format.

### SV_UpdateServerCommandsToClient
- **Signature:** `void SV_UpdateServerCommandsToClient(client_t *client, msg_t *msg)`
- **Purpose:** Retransmits all reliable server commands not yet acknowledged by the client.
- **Inputs:** Client, message buffer.
- **Outputs/Return:** Writes to `msg`; updates `client->reliableSent`.
- **Side effects:** Sets `client->reliableSent`.
- **Calls:** `MSG_WriteByte`, `MSG_WriteLong`, `MSG_WriteString`

### SV_SendMessageToClient
- **Signature:** `void SV_SendMessageToClient(msg_t *msg, client_t *client)`
- **Purpose:** Transmits a finalized message and updates `nextSnapshotTime` based on rate throttling.
- **Inputs:** Completed message, client.
- **Outputs/Return:** None; updates `client->nextSnapshotTime`, `client->rateDelayed`, frame metadata.
- **Side effects:** Calls `SV_Netchan_Transmit`. May call `Cvar_Set` (via `SV_RateMsec` if `sv_maxRate` is too low).
- **Calls:** `SV_Netchan_Transmit`, `SV_RateMsec`, `Sys_IsLANAddress`, `Cvar_Set` (inside `SV_RateMsec`)
- **Notes:** LAN/loopback clients always get `nextSnapshotTime = svs.time - 1` (immediate). Non-active clients get a minimum 1000 ms delay unless downloading.

### SV_SendClientSnapshot
- **Signature:** `void SV_SendClientSnapshot(client_t *client)`
- **Purpose:** Top-level per-client send: builds snapshot, writes reliable commands, writes snapshot data, appends download data, sends.
- **Calls:** `SV_BuildClientSnapshot`, `MSG_Init`, `MSG_WriteLong`, `SV_UpdateServerCommandsToClient`, `SV_WriteSnapshotToClient`, `SV_WriteDownloadToClient`, `MSG_Clear`, `SV_SendMessageToClient`
- **Notes:** Bots short-circuit after `SV_BuildClientSnapshot` (bots query snapshot state directly).

### SV_SendClientMessages
- **Signature:** `void SV_SendClientMessages(void)`
- **Purpose:** Per-frame driver that iterates all client slots and calls `SV_SendClientSnapshot` for those whose `nextSnapshotTime` has elapsed.
- **Notes:** Also handles sending of pending fragmented packets via `SV_Netchan_TransmitNextFragment`.

## Control Flow Notes
Called once per server frame from `SV_Frame` (in `sv_main.c`):
`SV_Frame → SV_SendClientMessages → SV_SendClientSnapshot → SV_BuildClientSnapshot + SV_WriteSnapshotToClient`.
`SV_BuildClientSnapshot` is the only function that mutates `sv.snapshotCounter` and writes into `svs.snapshotEntities[]`.

## External Dependencies
- **Includes:** `server.h` → `q_shared.h`, `qcommon.h`, `g_public.h`, `bg_public.h`
- **Defined elsewhere:**
  - `MSG_WriteDeltaEntity`, `MSG_WriteDeltaPlayerstate`, `MSG_WriteByte/Long/Bits/Data/String`, `MSG_Init`, `MSG_Clear` — `qcommon/msg.c`
  - `CM_PointLeafnum`, `CM_LeafArea`, `CM_LeafCluster`, `CM_ClusterPVS`, `CM_AreasConnected`, `CM_WriteAreaBits` — `qcommon/cm_*.c`
  - `SV_Netchan_Transmit`, `SV_Netchan_TransmitNextFragment` — `sv_net_chan.c`
  - `SV_GentityNum`, `SV_GameClientNum`, `SV_SvEntityForGentity` — `sv_game.c`
  - `SV_WriteDownloadToClient` — `sv_client.c`
  - `svs`, `sv`, `sv_padPackets`, `sv_maxRate`, `sv_lanForceRate`, `sv_maxclients` — globals/cvars
