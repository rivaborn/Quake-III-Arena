# code/client/cl_net_chan.c

## File Purpose
Provides the client-side network channel layer, wrapping the core `Netchan_*` functions with client-specific XOR obfuscation for outgoing and incoming game packets. It encodes transmitted messages and decodes received messages using a rolling key derived from the client challenge, server/sequence IDs, and acknowledged command strings.

## Core Responsibilities
- Encode outgoing client messages (bytes after `CL_ENCODE_START`) before transmission
- Decode incoming server messages (bytes after `CL_DECODE_START`) after reception
- Append `clc_EOF` marker before encoding and transmitting
- Delegate fragment transmission to the base `Netchan_TransmitNextFragment`
- Accumulate decoded byte counts in `newsize` for diagnostics/comparison with `oldsize`

## Key Types / Data Structures
None (no new types defined; uses `msg_t`, `netchan_t` from `qcommon.h`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `newsize` | `int` | global (extern linkage) | Accumulates total decoded bytes received; paired with `oldsize` (defined elsewhere) for bandwidth diagnostics |

## Key Functions / Methods

### CL_Netchan_Encode
- **Signature:** `static void CL_Netchan_Encode( msg_t *msg )`
- **Purpose:** XOR-obfuscates bytes `[CL_ENCODE_START, msg->cursize)` of an outgoing client message before it is sent over the network.
- **Inputs:** `msg` — the outgoing message buffer, which must contain at least 12 bytes of header (serverId, messageAcknowledge, reliableAcknowledge as 32-bit longs).
- **Outputs/Return:** None; modifies `msg->data` in-place.
- **Side effects:** Temporarily mutates and restores `msg->readcount`, `msg->bit`, `msg->oob` to safely call `MSG_ReadLong` three times without advancing the permanent read cursor.
- **Calls:** `MSG_ReadLong`
- **Notes:** Key is seeded from `clc.challenge ^ serverId ^ messageAcknowledge`; rotated per-byte using `clc.serverCommands[reliableAcknowledge & (MAX_RELIABLE_COMMANDS-1)]`. Non-ASCII or `'%'` characters in the command string are substituted with `'.'` to avoid injection hazards. Early-return if `msg->cursize <= CL_ENCODE_START` (12).

### CL_Netchan_Decode
- **Signature:** `static void CL_Netchan_Decode( msg_t *msg )`
- **Purpose:** XOR-decodes bytes `[msg->readcount + CL_DECODE_START, msg->cursize)` of an incoming server message after it has passed through `Netchan_Process`.
- **Inputs:** `msg` — the received message buffer whose first 4 bytes encode `reliableAcknowledge`.
- **Outputs/Return:** None; modifies `msg->data` in-place.
- **Side effects:** Temporarily mutates and restores `msg->readcount`, `msg->bit`, `msg->oob`.
- **Calls:** `MSG_ReadLong`, `LittleLong`
- **Notes:** Key seeded from `clc.challenge ^ LittleLong(*(unsigned *)msg->data)` (the netchan sequence number). Uses `clc.reliableCommands` (client-to-server commands) as the rotating key string. `CL_DECODE_START` is 4 (defined in `qcommon.h`).

### CL_Netchan_Transmit
- **Signature:** `void CL_Netchan_Transmit( netchan_t *chan, msg_t *msg )`
- **Purpose:** Appends EOF marker, encodes, then transmits a complete client message.
- **Inputs:** `chan` — the netchan to send on; `msg` — the message to send.
- **Outputs/Return:** None.
- **Side effects:** Writes `clc_EOF` byte into `msg`; encodes `msg->data`; calls `Netchan_Transmit` which sends a UDP packet.
- **Calls:** `MSG_WriteByte`, `CL_Netchan_Encode`, `Netchan_Transmit`

### CL_Netchan_TransmitNextFragment
- **Signature:** `void CL_Netchan_TransmitNextFragment( netchan_t *chan )`
- **Purpose:** Thin pass-through to send the next fragment of a fragmented message; no encryption applied to fragments (encryption happens at the full-message level before fragmentation).
- **Calls:** `Netchan_TransmitNextFragment`

### CL_Netchan_Process
- **Signature:** `qboolean CL_Netchan_Process( netchan_t *chan, msg_t *msg )`
- **Purpose:** Receives and reassembles an incoming packet, then decodes it.
- **Inputs:** `chan`, `msg`
- **Outputs/Return:** `qtrue` if the packet was valid and fully reassembled; `qfalse` otherwise.
- **Side effects:** Calls `CL_Netchan_Decode` on success; increments `newsize` by `msg->cursize`.
- **Calls:** `Netchan_Process`, `CL_Netchan_Decode`

## Control Flow Notes
This file sits entirely in the **per-frame network send/receive path**. `CL_Netchan_Transmit` is called from `CL_WritePacket` (cl_input.c) when sending user commands; `CL_Netchan_Process` is called from `CL_ParseServerMessage` / `CL_ReadPackets` (cl_parse.c / cl_input.c) when processing incoming data. Neither function participates in init or shutdown.

## External Dependencies
- `../game/q_shared.h` — base types (`byte`, `qboolean`, `msg_t` fields)
- `../qcommon/qcommon.h` — `msg_t`, `netchan_t`, `Netchan_Transmit`, `Netchan_TransmitNextFragment`, `Netchan_Process`, `MSG_ReadLong`, `MSG_WriteByte`, `LittleLong`, `CL_ENCODE_START`, `CL_DECODE_START`, `MAX_RELIABLE_COMMANDS`, `clc_EOF`
- `client.h` — `clc` (`clientConnection_t`: `challenge`, `serverCommands`, `reliableCommands`)
- `oldsize` — `extern int` defined elsewhere (likely `cl_parse.c`) used for bandwidth comparison
