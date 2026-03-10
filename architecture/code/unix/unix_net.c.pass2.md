# code/unix/unix_net.c — Enhanced Analysis

## Architectural Role

This file implements the Unix/Linux platform-specific network socket layer, fulfilling the `Sys_*` and `NET_*` function contract expected by the engine's qcommon core. It is the sole entry point for all UDP I/O on Unix systems and owns local interface enumeration, address classification, and socket lifecycle management. The file bridges the platform-agnostic network messaging layer (qcommon: `net_chan.c`, `msg.c`) to POSIX socket primitives, while remaining completely isolated from game logic and higher-level subsystems.

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon** (`net_chan.c`, common frame loop): calls `Sys_GetPacket` and `Sys_SendPacket` each frame to poll/flush UDP datagrams
- **server** (`sv_main.c`, `sv_init.c`): calls `NET_OpenIP`, `NET_Init`, `NET_Sleep`, `NET_Shutdown` during server lifecycle
- **client** (`cl_main.c`, `cl_net_chan.c`): calls `Sys_GetPacket`, `Sys_SendPacket`, `Sys_IsLANAddress` during connection and gameplay
- **game VM** and **cgame VM**: indirectly via cvar reads (`Cvar_Get`) for `net_noudp`, `net_ip`, `net_port`

### Outgoing (what this file depends on)
- **qcommon** (`qcommon.h`): `Cvar_Get`, `Cvar_SetValue`, `Com_Printf`, `Com_Error`, `NET_AdrToString`, `NET_ErrorString`, `PORT_SERVER` constant
- **q_shared.h**: `Com_sprintf`, `qboolean`, `byte` types; `netadr_t`, `cvar_t` struct definitions
- **POSIX socket API**: `socket()`, `sendto()`, `recvfrom()`, `bind()`, `ioctl()`, `setsockopt()`, `close()`
- **DNS/IP conversion**: `gethostbyname()`, `inet_addr()`, `inet_ntoa()`, `ntohl()`
- **Platform-specific ioctls** (macOS only): `SIOCGIFCONF`, `OSIOCGIFADDR` for interface enumeration

---

## Design Patterns & Rationale

### 1. Platform Abstraction Layer (Facade)
The file presents a narrow `Sys_*` and `NET_*` API to the engine core, hiding all POSIX socket details. This allows the same qcommon code to run unchanged on Windows (`win32/win_net.c`), Mac (`macosx/`), and headless systems (`null/`).

### 2. Type Bridge / Adapter Pattern
`NetadrToSockadr` and `SockadrToNetadr` convert bidirectionally between engine-level `netadr_t` (with type, IPv4 bytes, port) and POSIX `sockaddr_in`. This decouples the engine's address model from socket API dependencies.

### 3. Non-Blocking I/O with Event Loop Integration
Sockets are set non-blocking (`FIONBIO`) to integrate with the engine's frame-based event loop. `Sys_GetPacket` returns `qfalse` (not `EWOULDBLOCK`) so the caller can continue the frame; `NET_Sleep` uses `select()` for dedicated-server CPU yielding between frames.

### 4. Dual-Protocol Stub (Legacy IPX Support)
The code maintains `ipx_socket` and checks both IP and IPX paths in `Sys_GetPacket`/`Sys_SendPacket`, but IPX is never actually created on Unix (always 0). This is a compatibility ghost from Q3A's 1999 origin when IPX was still relevant; on modern Unix, it's dead code. The abstraction keeps platform-specific implementations symmetric.

### 5. Conditional Compilation for Platform Divergence
The `#ifdef MACOS_X` section in `NET_GetLocalAddress` shows two fundamentally different interface enumeration strategies:
- **macOS**: Uses `SIOCGIFCONF`/`OSIOCGIFADDR` ioctls to iterate AF_LINK + AF_INET pairs, skipping loopback.  
- **Generic POSIX**: Calls `gethostbyname()` on the machine hostname, relying on `/etc/hosts` or DNS.

This divergence reflects different OS capabilities; macOS's approach is more reliable on DHCP systems where hostname resolution may not match actual interfaces.

### 6. Global State Cache with Bounds
`localIP[MAX_IPS][4]` and `numIP` implement a simple, pre-allocated cache of local IPv4 addresses populated once at startup. This avoids repeated address enumeration during gameplay but is bound to 16 interfaces.

---

## Data Flow Through This File

### Initialization Phase (Engine Startup)
1. **`NET_Init()`** reads `net_noudp` cvar; if 0, calls `NET_OpenIP()`
2. **`NET_OpenIP()`** reads `net_ip` (bind address, default "localhost") and `net_port` (default `PORT_SERVER = 27960`)
3. **`NET_IPSocket()`** creates a UDP socket, configures non-blocking + broadcast, binds to the address:port
4. **`NET_GetLocalAddress()`** enumerates all non-loopback AF_INET interfaces on the machine (platform-divergent macOS vs. POSIX path) and populates `localIP[]`, `numIP`
5. If port is in use, `NET_OpenIP` tries up to 10 consecutive ports; fatal error if all fail
6. **`NET_Sleep()`** setup: dedicated servers will call this each frame to yield CPU

### Frame Loop (Per-Frame During Gameplay)
1. **Inbound**: Caller (qcommon, server, client) calls `Sys_GetPacket()` to drain one UDP datagram
   - Iterates `ip_socket` then `ipx_socket` (ipx_socket is always 0 on Unix, so only IP is tried)
   - `recvfrom()` reads into `net_message->data`
   - Converts sender's `sockaddr_in` → `netadr_t` via `SockadrToNetadr()`
   - Returns `qtrue` and the sender address; or `qfalse` if EWOULDBLOCK/ECONNREFUSED

2. **Outbound**: Caller constructs `netadr_t`, calls `Sys_SendPacket(length, data, to)`
   - `NetadrToSockadr()` converts address to `sockaddr_in`
   - `sendto()` dispatches the UDP packet
   - Prints error if send fails (but doesn't propagate error to caller)

3. **Address Classification**: Game/server calls `Sys_IsLANAddress(adr)` to check if an address is local (loopback, IPX, or same subnet as `localIP[]`), used for rate-limiting decisions
   - Compares IP address class (A, B, C) and octet patterns against RFC 1918 blocks (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)

4. **CPU Yield** (Dedicated Server): `NET_Sleep(msec)` blocks on `select()` waiting for data on `ip_socket` or stdin, with timeout, to avoid busy-spinning between frames

### Shutdown Phase
**`NET_Shutdown()`** closes `ip_socket` and nullifies it.

---

## Learning Notes

### 1. **IPv4-Only Hard Constraint**
The entire file is locked to IPv4: `AF_INET` hardcoded, no `AF_INET6` paths, IP addresses are always 4 bytes. IPv6 support would require:
- Expanding `netadr_t` to hold a 16-byte address + address-family tag
- Refactoring `localIP` and all address logic
- Dual-stack socket code paths (or separate IPv6 sockets)

This is a fundamental architectural limitation inherited from 1999; modern games would use a union or generic address type.

### 2. **Blocking DNS as a Performance Trap**
`Sys_StringToSockaddr()` calls `gethostbyname()`, which blocks the entire engine thread on network I/O if DNS is slow or unreachable. For a real-time game, this could cause hitches. Modern engines would use async DNS (getaddrinfo_a, resolver libraries, or a dedicated thread).

### 3. **Non-Blocking I/O Pattern Dependency**
This file assumes the caller has an event loop (select-based). If the caller tried to block on `Sys_GetPacket()`, the game would freeze. The API contract is implicit: "return quickly, we have other work to do."

### 4. **macOS vs. POSIX Dichotomy as an Idiosyncrasy**
The macOS `NET_GetLocalAddress` is significantly more complex (100+ lines) than the generic POSIX version (40 lines) because:
- macOS's `SIOCGIFCONF` returns both AF_LINK (hardware address) and AF_INET (IP address) entries; the code must match pairs by interface name and filter loopback by link-layer type.
- Generic POSIX assumes `gethostbyname()` will return the canonical hostname and all aliases; relies on static `/etc/hosts` or a well-configured nameserver.

This reflects the reality that "POSIX" is not monolithic; major Unix variants have idiosyncratic APIs.

### 5. **Dead Code: IPX Socket**
On Unix, `ipx_socket` is never created (always 0) because Unix networking is TCP/IP native. But the file still iterates both sockets in `Sys_GetPacket` and dispatches address types in `Sys_SendPacket`. This dead code costs almost nothing and maintains API symmetry with Windows/Mac ports, but is confusing to read.

### 6. **Error Handling Strategy: Fail Fast, Print Later**
- Socket creation fails → `Com_Error(ERR_FATAL)` immediately (engine cannot run without UDP)
- Packet send fails → printed warning but no propagation (game continues, packet is lost)
- Packet receive errors (except EWOULDBLOCK) → printed warning, packet dropped silently
- DNS resolution fails → return `qfalse` to caller (caller retries or treats as invalid address)

This is asymmetric but pragmatic: fatal errors halt the engine, transient errors log and continue.

### 7. **Local IP Enumeration: Cache-and-Forget**
`NET_GetLocalAddress()` is called once at startup and caches results in `localIP[]`. It never re-enumerates, so:
- If the machine's network config changes at runtime (interface added, DHCP renewal), the cache is stale
- `Sys_IsLANAddress()` uses the stale cache, potentially mis-classifying addresses
- For a game server, this is usually acceptable (uptime often spans network reconfig events)

### 8. **Signature Consistency: Platform Layer as a Facade**
All `Sys_*` and `NET_*` functions are thin wrappers adhering to an implicit contract:
- **No blocking calls** (except intentional `NET_Sleep()`)
- **No dynamic allocation** (fixed `localIP[]` buffer)
- **No dependency on game state** (pure platform abstraction)
- **No VM or scripting** (all calls are C-level, engine core only)

---

## Potential Issues

### 1. **Off-by-One Bug in POSIX `NET_GetLocalAddress()`** (lines ~520–530)
```c
numIP = 0;
while( ( p = hostInfo->h_addr_list[numIP++] ) != NULL && numIP < MAX_IPS ) {
    ip = ntohl( *(int *)p );
    localIP[ numIP ][0] = p[0];  // <-- stores at index numIP, not numIP-1
    ...
}
```
The loop increments `numIP` before storing, so the first address is stored at `localIP[1]`, not `localIP[0]`. `localIP[0]` is left zeroed, and if there are exactly `MAX_IPS` addresses, the last one is dropped. The macOS branch avoids this by incrementing *after* storing.

### 2. **No IPv6 Support**
Clients/servers are permanently IPv4-only. Modern networks increasingly use IPv6 or dual-stack.

### 3. **`MAX_IPS = 16` Hard Cap**
Machines with 16+ interfaces (VirtualBox, containers, bond interfaces) will silently drop IPs beyond the 16th. `Sys_IsLANAddress()` will miss traffic from those interfaces.

### 4. **Blocking `gethostbyname()` in Frame Path**
If the engine calls `Sys_StringToAdr()` (e.g., to join a server by hostname) during gameplay, a slow DNS lookup will block the entire frame, causing visible hitching. The first-pass doc notes this but doesn't flag it as an issue; it's a trap for callers.

### 5. **No Socket Error Logging Context**
`NET_ErrorString()` is called but its definition is not in this file (presumably in qcommon). If it returns a generic "Socket error," the caller has no context (which socket, which operation, errno value not propagated).

### 6. **`inet_addr()` Deprecated**
The file uses `inet_addr()` (deprecated since C99, marked obsolete in POSIX.1-2008). Modern code should use `inet_pton()` or `getaddrinfo()`.

---

**Summary**: This file is a textbook platform abstraction layer for a 1999-era game engine ported to Unix (2005). It's IPv4-only, assumes non-blocking event-driven I/O, and diverges on macOS for interface enumeration. The design is sound for its era and constraints, but shows its age (no IPv6, synchronous DNS, dead IPX code). The off-by-one bug in POSIX `NET_GetLocalAddress()` is likely unnoticed because most servers run on static configs with few interfaces.
