# code/qcommon/net_chan.c

## File Purpose
Implements the Quake III reliable sequenced network channel (`netchan_t`) layer, providing packet fragmentation/reassembly, out-of-order/duplicate suppression, and loopback routing. Also supplies address utility functions (`NET_CompareAdr`, `NET_AdrToString`, `NET_StringToAdr`) and out-of-band datagram helpers.

## Core Responsibilities
- Initialize and configure network channels (`Netchan_Init`, `Netchan_Setup`)
- Transmit messages, fragmenting payloads ≥ `FRAGMENT_SIZE` across multiple UDP packets
- Reassemble incoming fragments into a complete message buffer
- Discard duplicate and out-of-order packets; track dropped packet count
- Route loopback packets through in-process ring buffers instead of the OS socket
- Provide out-of-band text and binary datagram sending (`NET_OutOfBandPrint`, `NET_OutOfBandData`)
- Parse and format network addresses (`NET_StringToAdr`, `NET_AdrToString`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `loopmsg_t` | struct | Single loopback message slot: fixed-size `data` buffer + `datalen` |
| `loopback_t` | struct | Ring buffer of `MAX_LOOPBACK` (16) `loopmsg_t` slots with `get`/`send` cursors |

(All other key types — `netchan_t`, `msg_t`, `netadr_t` — are defined in `qcommon.h`.)

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `showpackets` | `cvar_t *` | global | Log all sent/received packet info to console |
| `showdrop` | `cvar_t *` | global | Log dropped/out-of-order packets to console |
| `qport` | `cvar_t *` | global | Client-side qport value embedded in every outgoing packet |
| `netsrcString` | `static char*[2]` | file-static | Human-readable labels `"client"` / `"server"` for logging |
| `loopbacks` | `loopback_t[2]` | global | Two loopback ring buffers, indexed by `netsrc_t` (client=0, server=1) |

## Key Functions / Methods

### Netchan_Init
- **Signature:** `void Netchan_Init( int port )`
- **Purpose:** Register CVars and set the initial `net_qport` value; called once at startup.
- **Inputs:** `port` — suggested qport (masked to 16 bits)
- **Outputs/Return:** void
- **Side effects:** Creates `showpackets`, `showdrop`, `qport` CVars
- **Calls:** `Cvar_Get`, `va`
- **Notes:** `port & 0xffff` prevents sign extension issues.

### Netchan_Setup
- **Signature:** `void Netchan_Setup( netsrc_t sock, netchan_t *chan, netadr_t adr, int qport )`
- **Purpose:** Zero-initialize a `netchan_t` and populate its fields for a new connection.
- **Inputs:** `sock` (NS_CLIENT/NS_SERVER), destination `adr`, `qport`
- **Outputs/Return:** void (writes into `*chan`)
- **Side effects:** Zeroes `*chan`; `outgoingSequence` starts at 1
- **Calls:** `Com_Memset`

### Netchan_TransmitNextFragment
- **Signature:** `void Netchan_TransmitNextFragment( netchan_t *chan )`
- **Purpose:** Send the next pending fragment from `chan->unsentBuffer`. Advances `unsentFragmentStart`; clears `unsentFragments` and increments `outgoingSequence` only after the final fragment (where `fragmentLength < FRAGMENT_SIZE`).
- **Inputs:** `chan` — channel with active unsent fragments
- **Outputs/Return:** void
- **Side effects:** Calls `NET_SendPacket`; mutates `chan->unsentFragmentStart`, `chan->outgoingSequence`, `chan->unsentFragments`
- **Calls:** `MSG_InitOOB`, `MSG_WriteLong`, `MSG_WriteShort`, `MSG_WriteData`, `NET_SendPacket`, `Com_Printf`
- **Notes:** A payload that is exactly `FRAGMENT_SIZE` bytes requires a second zero-length fragment to signal completion.

### Netchan_Transmit
- **Signature:** `void Netchan_Transmit( netchan_t *chan, int length, const byte *data )`
- **Purpose:** Send a message; if `length >= FRAGMENT_SIZE`, copy to `unsentBuffer` and kick off fragmented delivery via `Netchan_TransmitNextFragment`; otherwise send in one datagram.
- **Inputs:** `chan`, byte `data`, `length`
- **Outputs/Return:** void
- **Side effects:** `ERR_DROP` if `length > MAX_MSGLEN`; increments `outgoingSequence`; may call `NET_SendPacket`
- **Calls:** `Com_Error`, `Com_Memcpy`, `Netchan_TransmitNextFragment`, `MSG_InitOOB`, `MSG_WriteLong`, `MSG_WriteShort`, `MSG_WriteData`, `NET_SendPacket`, `Com_Printf`

### Netchan_Process
- **Signature:** `qboolean Netchan_Process( netchan_t *chan, msg_t *msg )`
- **Purpose:** Validate and process an incoming sequenced packet. Handles fragment reassembly, drops out-of-order packets, and returns `qtrue` only when a complete message is ready to consume.
- **Inputs:** `chan`, `msg` (incoming packet, must be `MAX_MSGLEN`-capable)
- **Outputs/Return:** `qtrue` if a full message is ready; `qfalse` if fragment is partial, out-of-order, or illegal
- **Side effects:** Updates `chan->incomingSequence`, `chan->dropped`, `chan->fragmentSequence`, `chan->fragmentLength`, `chan->fragmentBuffer`; rewrites `msg->data` on final fragment
- **Calls:** `MSG_BeginReadingOOB`, `MSG_ReadLong`, `MSG_ReadShort`, `Com_Printf`, `NET_AdrToString`, `Com_Memcpy`
- **Notes:** Fragment ordering is strict — if a fragment arrives out of order the whole sequence is reset and `qfalse` is returned; partially accumulated data is retained but the missing chunk must be re-received.

### NET_SendPacket
- **Signature:** `void NET_SendPacket( netsrc_t sock, int length, const void *data, netadr_t to )`
- **Purpose:** Dispatch a datagram: route to loopback ring, silently drop BOT/BAD destinations, or hand off to `Sys_SendPacket`.
- **Side effects:** Calls `NET_SendLoopPacket` or `Sys_SendPacket`; optional console print for OOB packets

### NET_OutOfBandPrint / NET_OutOfBandData
- `NET_OutOfBandPrint`: Prepends a 4-byte `0xFFFFFFFF` header to a `vsprintf`-formatted string and calls `NET_SendPacket`. **Note:** uses `vsprintf` with no length bound — potential overflow risk.
- `NET_OutOfBandData`: Similar, but compresses the payload with `Huff_Compress` at offset 12 before sending.

### NET_StringToAdr
- **Signature:** `qboolean NET_StringToAdr( const char *s, netadr_t *a )`
- **Purpose:** Parse a host[:port] string; traps `"localhost"` for loopback; delegates to `Sys_StringToAdr` for real addresses; validates broadcast sentinel `255.255.255.255`.
- **Calls:** `Sys_StringToAdr`, `Q_strncpyz`, `strstr`, `BigShort`, `atoi`

## Control Flow Notes
- **Init:** `Netchan_Init` is called once during `Com_Init`.
- **Per-frame (server/client):** Incoming packets enter `Netchan_Process` before game logic; outgoing messages go through `Netchan_Transmit`. If `chan->unsentFragments` is set after a transmit, the caller (e.g., `SV_SendClientGameState`) is responsible for draining fragments via repeated `Netchan_TransmitNextFragment` calls.
- **Loopback path:** Local listen-server uses `loopbacks[0/1]` ring buffers, bypassing the OS entirely.

## External Dependencies
- **Includes:** `../game/q_shared.h`, `qcommon.h`
- **Defined elsewhere:**
  - `MSG_*` — `code/qcommon/msg.c`
  - `Cvar_Get` — `code/qcommon/cvar.c`
  - `Sys_SendPacket`, `Sys_StringToAdr` — platform layer (`win32/`, `unix/`)
  - `Huff_Compress` — `code/qcommon/huffman.c`
  - `Com_Error`, `Com_Printf`, `Com_Memset`, `Com_Memcpy`, `Com_sprintf` — `code/qcommon/common.c`
  - `NET_Init`, `NET_Shutdown` — platform-specific net init (not in this file)
