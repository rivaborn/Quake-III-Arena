# code/qcommon/net_chan.c — Enhanced Analysis

## Architectural Role

`net_chan.c` is the **reliable transport boundary** in `qcommon`, sitting between raw OS sockets (`Sys_SendPacket`/`Sys_GetPacket` in `win32/`/`unix/`) and the high-level message parsers in `code/client/cl_parse.c` and `code/server/sv_client.c`. It provides the only sequenced, fragment-reassembled, out-of-order-filtered delivery guarantee in an otherwise pure UDP stack. Critically, it is *not* the outermost networking layer for in-game traffic: `code/client/cl_net_chan.c` and `code/server/sv_net_chan.c` each wrap the functions here with a challenge-derived XOR rolling key, so `net_chan.c` operates on cleartext messages while the wrappers handle obfuscation. Out-of-band connection traffic (challenge, connect, status) bypasses the `netchan_t` machinery entirely and is sent directly via `NET_OutOfBandPrint`.

## Key Cross-References

### Incoming (who depends on this file)

| Caller | How it uses net_chan.c |
|---|---|
| `code/client/cl_net_chan.c` | Wraps `Netchan_Transmit`/`Netchan_Process` with XOR obfuscation before forwarding to higher-level `cl_parse.c` |
| `code/server/sv_net_chan.c` | Mirror wrapper on the server side for encrypted in-game packets |
| `code/server/sv_client.c` | Calls `Netchan_Setup` on client connect; calls (via sv_net_chan) `Netchan_Transmit`/`Netchan_Process` each server frame |
| `code/server/sv_main.c` | Calls `NET_OutOfBandPrint` / `NET_OutOfBandData` for connectionless responses; calls `Netchan_Init` on startup |
| `code/client/cl_main.c` | Calls `Netchan_Init` / `Netchan_Setup` / `NET_StringToAdr` / `NET_OutOfBandPrint` during connection lifecycle |
| `code/client/cl_parse.c` | Consumes reassembled `msg_t` after `Netchan_Process` returns `qtrue` |

Globals `showpackets`, `showdrop`, and `qport` defined here are read only within this file (debug-diagnostic CVars, not shared by other subsystems).

### Outgoing (what this file depends on)

| Dependency | Purpose |
|---|---|
| `MSG_*` (`qcommon/msg.c`) | Bit-level read/write into `msg_t` for packet header fields |
| `Cvar_Get` (`qcommon/cvar.c`) | Register `showpackets`, `showdrop`, `net_qport` |
| `Sys_SendPacket` (platform layer) | Actual OS UDP write for non-loopback, non-bot destinations |
| `Sys_StringToAdr` (platform layer) | DNS/IP resolution in `NET_StringToAdr` |
| `Huff_Compress` (`qcommon/huffman.c`) | Payload compression in `NET_OutOfBandData` |
| `Com_Error`/`Com_Printf`/`Com_Memset`/`Com_Memcpy` (`qcommon/common.c`) | Error reporting, console logging, memory operations |
| `NET_AdrToString`/`NET_CompareAdr` (this file, also used externally) | Self-referential: address utilities called within the same file by `Netchan_Process` |

## Design Patterns & Rationale

**Layered security-by-obscurity (abandoned):** The `#if 0`-guarded `Netchan_ScramblePacket` / `Netchan_UnScramblePacket` functions reveal that packet scrambling was attempted and then explicitly abandoned with the comment "A probably futile attempt." The XOR obfuscation that *was* retained moved up the stack to `cl_net_chan.c`/`sv_net_chan.c`, using a challenge-derived key rather than a seed derived from plaintext header fields — a deliberate security improvement, though still not cryptographically robust.

**Dual-indexed ring buffer for loopback:** `loopbacks[2]` indexed by `netsrc_t` (client=0, server=1) means a listen-server can operate with zero OS socket overhead. The pattern of using an enum value directly as an array index is idiomatic to this codebase's flat-C style — no abstraction overhead, but tightly coupled to `netsrc_t` staying a 2-value enum.

**High-bit sequence encoding:** Encoding the fragment flag in bit 31 of the sequence number (`FRAGMENT_BIT = 1<<31`) avoids adding a separate protocol field, preserving packet header compactness. The receiver strips the bit before using the sequence as a counter. This is a classic space-optimization tradeoff for a 1999-era UDP protocol where every byte in the header cost latency.

**qport NAT workaround:** The `qport` field exists purely to compensate for routers that remap source ports mid-session. Rather than redesigning the protocol, the server identifies clients by `(base_IP, qport)` instead of `(IP, port)`. This is a pragmatic fix that predates STUN/ICE by years.

## Data Flow Through This File

```
[Outgoing]
Game/Server logic
  → Netchan_Transmit(chan, len, data)
      if len >= FRAGMENT_SIZE:
          copy to chan->unsentBuffer
          → Netchan_TransmitNextFragment (called repeatedly by caller)
      else:
          write 4-byte sequence + optional 2-byte qport header
          → NET_SendPacket(sock, len, buf, remoteAddress)
              if NA_LOOPBACK: → NET_SendLoopPacket (ring buffer write)
              else:           → Sys_SendPacket (OS UDP write)

[Incoming]
Sys_GetPacket → raw UDP datagram
  → cl_net_chan.c / sv_net_chan.c: XOR de-obfuscate
  → Netchan_Process(chan, msg)
      read sequence, qport, fragment fields from header
      if out-of-order → return qfalse
      if fragmented:
          accumulate into chan->fragmentBuffer
          if not final fragment → return qfalse
          reassemble: overwrite msg->data with complete message
      update chan->incomingSequence, chan->dropped
      → return qtrue (full message ready)
  → cl_parse.c / sv_client.c: parse game protocol
```

Key state transitions: `chan->outgoingSequence` advances only when a complete (possibly multi-fragment) message finishes sending. `chan->incomingSequence` advances only when a complete message is accepted. The `dropped` field is a *diagnostic counter*, not a retransmit trigger — Q3 uses an application-level reliable command channel layered on top, not TCP-style retransmission here.

## Learning Notes

**No retransmission here — reliability is above this layer.** Unlike TCP, `net_chan.c` does not retry dropped packets. The Q3 protocol instead maintains a separate "reliable command" queue in the game protocol layer (parsed in `cl_parse.c` / `sv_client.c`) that retransmits unacknowledged commands. This file only guarantees ordering and fragment reassembly for packets that do arrive.

**Fragment completion edge case:** A payload of exactly `FRAGMENT_SIZE` bytes requires a second zero-byte fragment to signal completion. This is an explicit design decision documented in comments — the kind of subtle protocol corner case modern developers often forget and that causes interoperability bugs.

**Idiomatic era patterns:** The `static char s[64]` return value in `NET_AdrToString` (a shared static buffer, overwritten on each call) is characteristic of late-1990s C: fast, no allocation, but not reentrant and not safe to hold two addresses simultaneously. Modern engines would return `std::string` or require a caller-supplied buffer.

**`vsprintf` without bounds** in `NET_OutOfBandPrint` is a latent buffer overflow — the destination is a stack-allocated `MAX_MSGLEN` byte array, but the format string comes from callers who may exceed this. In a server environment this is reachable from the network.

**Connection to modern concepts:** The `netchan_t` prefigures what QUIC and DTLS provide at the OS/library level — ordered delivery, connection identity across IP:port changes, and a fragmentation layer — but implemented entirely in application-space C in ~700 lines.

## Potential Issues

- **`NET_OutOfBandPrint` buffer overflow:** `vsprintf` into a `MAX_MSGLEN`-bounded stack buffer with no length check. Callers from `sv_main.c` pass format strings derived from client-provided data (e.g., rcon responses), making this a plausible exploit surface in unpatched servers.
- **Fragment drop silently retains partial data:** When an out-of-order fragment is received (`fragmentStart != chan->fragmentLength`), the comment notes "we can still keep the part that we have so far." This means `chan->fragmentLength` is not reset on a missed fragment, only on a sequence change. If a packet is dropped mid-stream and a new sequence begins, the old partial buffer is silently discarded but its length field lingers until the next sequence-mismatch resets it — non-obvious state for a maintainer to reason about.
