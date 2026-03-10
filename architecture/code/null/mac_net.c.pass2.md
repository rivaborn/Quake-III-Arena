# code/null/mac_net.c — Enhanced Analysis

## Architectural Role

This file implements the platform abstraction layer's networking boundary for null/headless Q3A builds. It sits at the lowest level of the networking stack, beneath `qcommon/net_chan.c` (reliable UDP channels) and above any OS socket layer. By providing inert stub implementations, it allows the engine to compile and run without requiring platform-specific socket support—suitable for dedicated servers, headless tool runs, or early-stage ports. The real implementations exist in `code/win32/win_net.c` (Winsock), `code/unix/unix_net.c` (BSD sockets), and `code/macosx/` (CoreFoundation/CFSocket); this file defines the contract all platforms must honor.

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon/net_chan.c** (`Netchan_TransmitNextFragment`, `Netchan_Process`) — calls `Sys_SendPacket` to flush reliable UDP packets
- **qcommon/common.c** (`Com_Frame`) — drives per-frame network I/O via the main loop
- **client/cl_main.c** → **client/cl_net_chan.c** (`CL_SendPacket`) — routes usercmd packets to `Sys_SendPacket`
- **server/sv_main.c** → **server/sv_net_chan.c** (`SV_SendPacket`) — routes snapshot packets to `Sys_SendPacket`
- **qcommon/common.c** (`Com_Frame`) — calls `Sys_GetPacket` to poll for inbound data
- **client/cl_main.c** (`CL_PacketEvent`), **server/sv_main.c** (`SV_PacketEvent`) — dispatch inbound packets

### Outgoing (what this file depends on)
- **game/q_shared.h** — `qboolean` type, `strcmp`, `memset` via C stdlib includes
- **qcommon/qcommon.h** — type declarations for `netadr_t` (address struct with `type` field), `msg_t` (message buffer)
- C stdlib — `strcmp`, `memset`

## Design Patterns & Rationale

**Platform Abstraction Layer**: The engine separates networking from core logic via platform-specific implementations of three functions. Each platform (Win32, Unix, macOS, null) provides its own `NET_StringToAdr`, `Sys_SendPacket`, `Sys_GetPacket`. This decoupling allows:
- Single engine codebase compiled against different I/O subsystems
- Headless/null builds that skip socket code entirely (reduces binary size, dependency surface)
- Easy porting to new platforms: implement these three functions, recompile

**Null Object Pattern**: Instead of conditional compilation (`#ifdef NO_NETWORK`), this file provides silent no-ops. The engine logic never changes; only I/O is disabled. `Sys_SendPacket` discards data without error reporting; `Sys_GetPacket` always reports "no packet available."

## Data Flow Through This File

**Outbound path:**
1. Game (`server/sv_game.c`) calls `trap_DropClient`, triggering snapshots
2. Server (`sv_snapshot.c`) builds delta-encoded entityState/playerState
3. Network channel (`net_chan.c`) fragments and sequences the message
4. **`Sys_SendPacket(length, data, to)` is called** → silently discarded in this stub
5. No UDP datagram ever reaches the network

**Inbound path:**
1. Main loop (`common.c` `Com_Frame`) periodically calls **`Sys_GetPacket()`**
2. Returns `false` (no packet available)
3. Client and server expect no inbound messages
4. Network state machine (`net_chan.c`) times out waiting for acknowledgments

**Address resolution:**
- `NET_StringToAdr("localhost", &a)` → sets `a->type = NA_LOOPBACK`, returns `true`
- `NET_StringToAdr("192.168.1.1", &a)` → returns `false` (cannot parse IP addresses)
- Prevents any non-loopback connections (no multiplayer, no server discovery)

## Learning Notes

**Porting & Minimalism**: This file demonstrates how Q3A's architecture supports minimal I/O implementations. You could ship a single-player-only binary with only loopback networking, no socket library dependency. Compare with monolithic engines where network I/O is baked into core simulation.

**Contract vs. Implementation**: The three functions define Q3A's platform contract. Real implementations handle:
- IPv4/IPv6 address parsing (DNS resolution, dotted-quad parsing, port parsing)
- Non-blocking socket `send()` with error handling and retries
- Non-blocking socket `recv()` with fragment reassembly and out-of-order buffering
- Platform-specific address structures (`sockaddr_in` vs. `sockaddr_in6` vs. custom)

This stub implements the _minimum contract_: loopback only, fire-and-forget sending, no receiving.

**Idiomatic Era (2004)**: Unlike modern engines, Q3A hardcodes the platform abstraction as a set of C functions rather than a virtual-interface vtable. No factory pattern, no traits. This is typical of pre-2000s game engines, now considered rigid by modern standards (though perfectly functional).

## Potential Issues

- **Silent failures**: `Sys_SendPacket` doesn't return status or log warnings. A real network error would go undiagnosed.
- **No error propagation**: `NET_StringToAdr` returns only `qboolean`; can't distinguish "not a valid address" from "feature not supported."
- **Loopback only**: Disables all multiplayer and client-server play, suitable only for headless/testing scenarios.
