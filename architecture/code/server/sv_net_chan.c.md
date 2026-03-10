# code/server/sv_net_chan.c

## File Purpose
Provides server-side network channel wrapper functions that layer XOR-based obfuscation encoding/decoding on top of the base `Netchan` fragmentation and sequencing layer. It also manages a per-client outgoing message queue to prevent UDP packet bursts when large fragmented messages collide during transmission.

## Core Responsibilities
- XOR-encode outgoing server messages using client challenge, sequence number, and acknowledged command strings as a rolling key
- XOR-decode incoming client messages using matching key material
- Queue outgoing messages when the netchan already has unsent fragments, ensuring correct ordering
- Drain the outgoing queue by encoding and transmitting the next queued message once fragmentation completes
- Wrap `Netchan_Process` with a decode step for all received client packets

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `netchan_buffer_t` | struct (defined in `server.h`) | Linked-list node holding a buffered outgoing `msg_t` and its raw byte storage; used for the per-client transmit queue |
| `client_t` | struct (defined in `server.h`) | Per-client state including challenge, netchan, reliable command history, and the queue head/tail pointers |
| `msg_t` | struct (defined in `qcommon.h`) | Network message buffer with read/write cursors and bit-level state |

## Global / File-Static State
None.

## Key Functions / Methods

### SV_Netchan_Encode
- **Signature:** `static void SV_Netchan_Encode( client_t *client, msg_t *msg )`
- **Purpose:** XOR-obfuscates bytes `[SV_ENCODE_START, msg->cursize)` of an outgoing message in-place.
- **Inputs:** `client` — source of challenge, outgoing sequence, and `lastClientCommandString`; `msg` — message buffer to encode.
- **Outputs/Return:** void; modifies `msg->data` in-place.
- **Side effects:** Temporarily mutates `msg->readcount`, `msg->bit`, and `msg->oob` to read the leading `reliableAcknowledge` long, then restores them.
- **Calls:** `MSG_ReadLong`
- **Notes:** Key is seeded as `client->challenge ^ client->netchan.outgoingSequence`, then rolled per byte using the last received client command string. Characters `>127` or `'%'` are replaced with `'.'` to avoid format-string and high-bit issues in the key stream.

### SV_Netchan_Decode
- **Signature:** `static void SV_Netchan_Decode( client_t *client, msg_t *msg )`
- **Purpose:** XOR-decodes bytes starting at `msg->readcount + SV_DECODE_START` of a received client message in-place.
- **Inputs:** `client` — source of challenge and `reliableCommands` table; `msg` — already netchan-processed incoming buffer.
- **Outputs/Return:** void; modifies `msg->data` in-place.
- **Side effects:** Temporarily clears `msg->oob` to read the three leading longs (`serverId`, `messageAcknowledge`, `reliableAcknowledge`), then restores saved state.
- **Calls:** `MSG_ReadLong` (×3)
- **Notes:** Key is `client->challenge ^ serverId ^ messageAcknowledge`; the rolling component is the server's reliable command at index `reliableAcknowledge & (MAX_RELIABLE_COMMANDS-1)`.

### SV_Netchan_TransmitNextFragment
- **Signature:** `void SV_Netchan_TransmitNextFragment( client_t *client )`
- **Purpose:** Advances fragmented transmission; when the last fragment drains, pops and sends the next queued message.
- **Inputs:** `client` — target client.
- **Outputs/Return:** void.
- **Side effects:** Calls `Netchan_TransmitNextFragment`; may call `SV_Netchan_Encode`, `Netchan_Transmit`, updates `client->netchan_start_queue` and `client->netchan_end_queue`, calls `Z_Free` on the dequeued buffer node.
- **Calls:** `Netchan_TransmitNextFragment`, `SV_Netchan_Encode`, `Netchan_Transmit`, `Com_Error`, `Com_DPrintf`, `Z_Free`
- **Notes:** Triggers `ERR_DROP` if `netchan_end_queue` is NULL when the queue should be valid — a defensive sanity check.

### SV_Netchan_Transmit
- **Signature:** `void SV_Netchan_Transmit( client_t *client, msg_t *msg )`
- **Purpose:** Primary send path: appends `svc_EOF`, then either encodes and transmits immediately or queues if fragments are still pending.
- **Inputs:** `client` — destination; `msg` — fully composed outgoing message.
- **Outputs/Return:** void.
- **Side effects:** Writes `svc_EOF` to `msg`; may allocate a `netchan_buffer_t` via `Z_Malloc` and enqueue it; calls `MSG_Copy`, `Netchan_TransmitNextFragment` or `SV_Netchan_Encode`+`Netchan_Transmit`.
- **Calls:** `MSG_WriteByte`, `Z_Malloc`, `MSG_Copy`, `Netchan_TransmitNextFragment`, `SV_Netchan_Encode`, `Netchan_Transmit`, `Com_DPrintf`
- **Notes:** Messages must **not** be encoded before queuing because encoding depends on `outgoingSequence`, which is not yet finalized at queue time.

### SV_Netchan_Process
- **Signature:** `qboolean SV_Netchan_Process( client_t *client, msg_t *msg )`
- **Purpose:** Receives and validates a client packet through the base netchan layer, then decodes it.
- **Inputs:** `client`, `msg` — raw incoming packet.
- **Outputs/Return:** `qtrue` on success, `qfalse` if `Netchan_Process` rejects the packet (duplicate, out-of-order, etc.).
- **Side effects:** Calls `Netchan_Process` (updates `client->netchan` sequence state), then `SV_Netchan_Decode`.
- **Calls:** `Netchan_Process`, `SV_Netchan_Decode`

## Control Flow Notes
- **Per-frame send path:** `SV_SendMessageToClient` → `SV_Netchan_Transmit` (fragment queueing or direct send) → `SV_Netchan_TransmitNextFragment` called on subsequent frames until fragments drain, then queue is popped.
- **Per-frame receive path:** `SV_PacketEvent` → `SV_ExecuteClientMessage` → `SV_Netchan_Process` → upper-layer message parsing.
- No participation in init or shutdown phases.

## External Dependencies
- `../game/q_shared.h` — base types (`byte`, `qboolean`, `msg_t` primitives)
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `Netchan_Transmit`, `Netchan_TransmitNextFragment`, `Netchan_Process`, `MSG_ReadLong`, `MSG_WriteByte`, `MSG_Copy`, `Z_Malloc`, `Z_Free`, `Com_DPrintf`, `Com_Error`; constants `SV_ENCODE_START`, `SV_DECODE_START`, `MAX_RELIABLE_COMMANDS`, `svc_EOF`
- `server.h` — `client_t`, `netchan_buffer_t`, `MAX_MSGLEN`
- `Netchan_*` functions — defined in `qcommon/net_chan.c` (not this file)
