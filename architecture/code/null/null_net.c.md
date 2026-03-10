# code/null/null_net.c

## File Purpose
Provides a null (stub) implementation of the platform-specific networking layer for Quake III Arena. It is used in headless or minimal build configurations where real network I/O is not needed, implementing only loopback address resolution.

## Core Responsibilities
- Stub out `Sys_SendPacket` so packet transmission is a no-op
- Stub out `Sys_GetPacket` so packet reception always returns nothing
- Implement `NET_StringToAdr` with minimal support: only resolves `"localhost"` to `NA_LOOPBACK`; all other addresses fail

## Key Types / Data Structures
None defined in this file. Uses types from `qcommon.h`:

| Name | Kind | Purpose |
|------|------|---------|
| `netadr_t` | struct | Network address (type, IP, IPX, port) |
| `netadrtype_t` | enum | Address type tag (`NA_LOOPBACK`, `NA_IP`, etc.) |
| `msg_t` | struct | Network message buffer with read/write state |

## Global / File-Static State
None.

## Key Functions / Methods

### NET_StringToAdr
- **Signature:** `qboolean NET_StringToAdr(char *s, netadr_t *a)`
- **Purpose:** Converts a string address to a `netadr_t`. Only handles `"localhost"`; all other inputs return false.
- **Inputs:** `s` — address string; `a` — output address struct
- **Outputs/Return:** `qtrue` if resolved, `qfalse` otherwise
- **Side effects:** Zeroes `*a` and sets `a->type = NA_LOOPBACK` on match
- **Calls:** `strcmp`, `memset`
- **Notes:** Real platforms (Unix/Win32) implement full DNS/IP parsing here. This stub intentionally omits it.

### Sys_SendPacket
- **Signature:** `void Sys_SendPacket(int length, void *data, netadr_t to)`
- **Purpose:** No-op stub for transmitting a UDP packet.
- **Inputs:** `length`, `data`, `to` — all ignored
- **Outputs/Return:** void
- **Side effects:** None
- **Calls:** None
- **Notes:** Signature diverges slightly from `qcommon.h` (`const void *data` vs `void *data`).

### Sys_GetPacket
- **Signature:** `qboolean Sys_GetPacket(netadr_t *net_from, msg_t *net_message)`
- **Purpose:** No-op stub for receiving a UDP packet from the system event queue.
- **Inputs:** `net_from`, `net_message` — ignored
- **Outputs/Return:** Always `qfalse`
- **Side effects:** None
- **Calls:** None
- **Notes:** Comment explicitly states it is never called by game logic, only by the system event queue.

## Control Flow Notes
This file contributes to startup and per-frame network polling. `Sys_GetPacket` would normally be called during `Com_EventLoop` to drain incoming packets; here it is a permanent no-op. `NET_StringToAdr` is called during connection setup. There is no shutdown path needed since no sockets are opened.

## External Dependencies
- `../qcommon/qcommon.h` — provides `netadr_t`, `netadrtype_t`, `msg_t`, `qboolean`, `NA_LOOPBACK`, and the declared signatures for `NET_StringToAdr`, `Sys_SendPacket`, and `Sys_GetPacket`
- `strcmp`, `memset` — C standard library (via `qcommon.h` transitively including `q_shared.h`)
- `Sys_SendPacket`, `Sys_GetPacket`, `NET_StringToAdr` — declared in `qcommon.h`; **defined here** as null implementations
