# code/null/mac_net.c

## File Purpose
A null/stub implementation of the Mac network layer for Quake III Arena. It provides non-functional placeholder implementations of the platform-specific network functions required by the engine, always returning failure or doing nothing.

## Core Responsibilities
- Provide a stub `NET_StringToAdr` that only resolves "localhost" to `NA_LOOPBACK`
- Provide a no-op `Sys_SendPacket` that discards all outgoing packet data
- Provide a stub `Sys_GetPacket` that always reports no incoming packets

## Key Types / Data Structures
None defined in this file; uses `netadr_t` and `msg_t` from `qcommon.h`.

## Global / File-Static State
None.

## Key Functions / Methods

### NET_StringToAdr
- **Signature:** `qboolean NET_StringToAdr(char *s, netadr_t *a)`
- **Purpose:** Converts a network address string to a `netadr_t`. Only handles the special case `"localhost"`; all other addresses fail.
- **Inputs:** `s` — address string; `a` — output address struct
- **Outputs/Return:** `qtrue` if `s` is `"localhost"` (sets `a->type = NA_LOOPBACK`); `qfalse` otherwise
- **Side effects:** Zeroes `*a` and sets `a->type` on success
- **Calls:** `strcmp`, `memset`
- **Notes:** Real platforms (Win32, Linux) would parse IP/IPX addresses here. This stub is intentionally minimal.

### Sys_SendPacket
- **Signature:** `void Sys_SendPacket(int length, void *data, netadr_t to)`
- **Purpose:** No-op stub for sending a UDP packet. Silently discards the data.
- **Inputs:** `length`, `data`, `to` — all ignored
- **Outputs/Return:** void
- **Side effects:** None
- **Calls:** None
- **Notes:** On real platforms this would call the OS socket send API.

### Sys_GetPacket
- **Signature:** `qboolean Sys_GetPacket(netadr_t *net_from, msg_t *net_message)`
- **Purpose:** No-op stub for receiving a UDP packet. Always reports no packet available.
- **Inputs:** `net_from`, `net_message` — both ignored
- **Outputs/Return:** Always `qfalse`
- **Side effects:** None
- **Calls:** None
- **Notes:** Comment states it is "never called by game logic, just the system event queuing," consistent with its null role here.

## Control Flow Notes
These functions are called from the engine's network layer (`net_chan.c`, `common.c`) during the frame event loop. Because all three stubs are inert, this file effectively disables network I/O, making it suitable only for headless/null builds or compile-time stubs on platforms not yet supported.

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `netadr_t`, `netadrtype_t` (`NA_LOOPBACK`), `memset`
- `../qcommon/qcommon.h` — `msg_t`, `netadr_t`, `Sys_SendPacket`/`Sys_GetPacket` declarations
- `strcmp`, `memset` — C standard library (via `q_shared.h` includes)
