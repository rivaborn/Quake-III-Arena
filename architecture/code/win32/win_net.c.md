# code/win32/win_net.c

## File Purpose
Windows-specific (Winsock) implementation of the low-level network layer for Quake III Arena. It creates and manages UDP sockets for IP and IPX protocols, handles SOCKS5 proxy tunneling, and provides packet send/receive primitives consumed by the platform-independent `qcommon` network layer.

## Core Responsibilities
- Initialize and shut down the Winsock library (`WSAStartup`/`WSACleanup`)
- Open, configure, and close UDP sockets for IP (`ip_socket`) and IPX (`ipx_socket`) protocols
- Implement optional SOCKS5 proxy negotiation and UDP-associate relay
- Convert between engine `netadr_t` and OS `sockaddr`/`sockaddr_ipx` representations
- Receive incoming packets (`Sys_GetPacket`) and send outgoing packets (`Sys_SendPacket`)
- Classify remote addresses as LAN or WAN (`Sys_IsLANAddress`)
- Enumerate and cache local IP addresses for LAN detection

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `netadr_t` | struct (defined in `qcommon.h`) | Engine-level network address (type + IP/IPX bytes + port) |
| `msg_t` | struct (defined in `qcommon.h`) | Network message buffer with read/write cursors |
| `cvar_t` | struct (defined in `q_shared.h`) | Console variable holding runtime configuration values |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `winsockdata` | `WSADATA` | static | Winsock library metadata filled by `WSAStartup` |
| `winsockInitialized` | `qboolean` | static | Guards double-init and shutdown-before-init |
| `usingSocks` | `qboolean` | static | Enables SOCKS5 relay path in send/receive |
| `networkingEnabled` | `qboolean` | static | Tracks current enabled state to avoid redundant reconfig |
| `ip_socket` | `SOCKET` | static | Bound UDP/IP socket |
| `socks_socket` | `SOCKET` | static | TCP connection to SOCKS5 server |
| `ipx_socket` | `SOCKET` | static | Bound UDP/IPX socket |
| `socksRelayAddr` | `struct sockaddr` | static | Address of the SOCKS5 relay endpoint for UDP |
| `localIP[MAX_IPS][4]` | `byte` array | static | Cached local interface IPs for LAN classification |
| `numIP` | `int` | static | Number of valid entries in `localIP` |
| `net_noudp`, `net_noipx` | `cvar_t *` | static | Disable respective protocol sockets |
| `net_socksEnabled/Server/Port/Username/Password` | `cvar_t *` | static | SOCKS5 proxy configuration cvars |
| `recvfromCount` | `int` | global | Performance counter incremented each `recvfrom` call |
| `socksBuf[4096]` | `char` | static | Scratch buffer for SOCKS5-wrapped outgoing packets |

## Key Functions / Methods

### NET_Init
- **Signature:** `void NET_Init(void)`
- **Purpose:** Bootstrap Winsock and start networking.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `winsockInitialized`; registers cvars; calls `NET_Config(qtrue)` to open sockets.
- **Calls:** `WSAStartup`, `NET_GetCvars`, `NET_Config`
- **Notes:** Called once at engine startup from `win_main.c` / common init.

### NET_Shutdown
- **Signature:** `void NET_Shutdown(void)`
- **Purpose:** Close all sockets and tear down Winsock.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Closes all open sockets; calls `WSACleanup`; clears `winsockInitialized`.
- **Calls:** `NET_Config`, `WSACleanup`

### NET_Config
- **Signature:** `void NET_Config(qboolean enableNetworking)`
- **Purpose:** Central socket lifecycle manager; opens or closes sockets based on enable state and cvar changes.
- **Inputs:** `enableNetworking` — desired enabled state
- **Outputs/Return:** None
- **Side effects:** Closes existing sockets on `stop`; calls `NET_OpenIP`/`NET_OpenIPX` on `start`; updates `networkingEnabled`.
- **Calls:** `NET_GetCvars`, `closesocket`, `NET_OpenIP`, `NET_OpenIPX`
- **Notes:** Idempotent when state and cvars are unchanged.

### Sys_GetPacket
- **Signature:** `qboolean Sys_GetPacket(netadr_t *net_from, msg_t *net_message)`
- **Purpose:** Non-blocking poll for one incoming UDP packet from either IP or IPX socket.
- **Inputs:** Output pointers for sender address and message buffer.
- **Outputs/Return:** `qtrue` if a packet was received and stored in `net_message`.
- **Side effects:** Increments `recvfromCount`; strips SOCKS5 header if relay match detected.
- **Calls:** `recvfrom`, `WSAGetLastError`, `SockadrToNetadr`, `NET_AdrToString`, `Com_Printf`
- **Notes:** Called by the system event loop, not game logic. Oversize packets are silently dropped.

### Sys_SendPacket
- **Signature:** `void Sys_SendPacket(int length, const void *data, netadr_t to)`
- **Purpose:** Send a UDP datagram to the given address.
- **Inputs:** Packet length, data pointer, destination `netadr_t`.
- **Outputs/Return:** None
- **Side effects:** If `usingSocks` and destination is NA_IP, wraps data in a SOCKS5 UDP header into `socksBuf` and redirects to `socksRelayAddr`.
- **Calls:** `NetadrToSockadr`, `sendto`, `WSAGetLastError`, `NET_ErrorString`, `Com_Printf`
- **Notes:** `WSAEWOULDBLOCK` and broadcast-related `WSAEADDRNOTAVAIL` are silently ignored.

### NET_OpenSocks
- **Signature:** `void NET_OpenSocks(int port)`
- **Purpose:** Establish a SOCKS5 TCP session and perform UDP-associate negotiation.
- **Inputs:** Local UDP port to associate.
- **Outputs/Return:** None
- **Side effects:** Opens `socks_socket`; populates `socksRelayAddr`; sets `usingSocks = qtrue` on success.
- **Calls:** `socket`, `gethostbyname`, `connect`, `send`, `recv`, `Com_Printf`
- **Notes:** Supports both anonymous and RFC 1929 username/password auth. Uses blocking I/O; only called at init time.

### NET_IPSocket
- **Signature:** `int NET_IPSocket(char *net_interface, int port)`
- **Purpose:** Create, configure, and bind a non-blocking broadcast-capable UDP/IP socket.
- **Inputs:** Interface name/IP string (or NULL for `INADDR_ANY`), port number (`PORT_ANY` for ephemeral).
- **Outputs/Return:** `SOCKET` handle, or 0 on failure.
- **Calls:** `socket`, `ioctlsocket`, `setsockopt`, `Sys_StringToSockaddr`, `bind`, `closesocket`

### NET_IPXSocket
- **Signature:** `int NET_IPXSocket(int port)`
- **Purpose:** Create and bind a non-blocking IPX socket.
- **Inputs:** Port number.
- **Outputs/Return:** `SOCKET` handle, or 0 on failure.

### Sys_IsLANAddress
- **Signature:** `qboolean Sys_IsLANAddress(netadr_t adr)`
- **Purpose:** Determine if an address belongs to the local network (for rate-limiting bypass).
- **Inputs:** Address to test.
- **Outputs/Return:** `qtrue` for loopback, IPX, class-A/B/C matches against `localIP[]`, and RFC 1918 ranges.
- **Notes:** Does not handle IPv6; class detection uses `adr.ip[0]` high-bit patterns.

### NetadrToSockadr / SockadrToNetadr
- Convert bidirectionally between engine `netadr_t` and OS `sockaddr`/`sockaddr_ipx`. Handle NA_IP, NA_IPX, NA_BROADCAST, NA_BROADCAST_IPX.

### Sys_StringToSockaddr / Sys_StringToAdr
- Parse a hostname or dotted-decimal string (or 21-char IPX hex string) into a `sockaddr` / `netadr_t`. Uses `inet_addr` for numeric IPv4, `gethostbyname` for hostnames.

## Control Flow Notes
- **Init:** `NET_Init` → `WSAStartup` → `NET_GetCvars` → `NET_Config(qtrue)` → `NET_OpenIP` / `NET_OpenIPX`.
- **Per-frame:** `Sys_GetPacket` is polled by the system event queue loop (`Com_EventLoop`) each frame; `Sys_SendPacket` is called by the higher-level `NET_SendPacket` wrapper in `qcommon`.
- **Shutdown:** `NET_Shutdown` → `NET_Config(qfalse)` closes sockets → `WSACleanup`.
- **Restart:** `NET_Restart` re-invokes `NET_Config` with the current `networkingEnabled` state to apply latched cvar changes.

## External Dependencies
- `<winsock.h>`, `<wsipx.h>` — Winsock and IPX socket APIs (via `win_local.h`)
- `../game/q_shared.h` — `qboolean`, `byte`, `cvar_t`, `netadr_t` type definitions
- `../qcommon/qcommon.h` — `msg_t`, `NET_AdrToString`, `Com_Printf`, `Com_Error`, `Cvar_Get`, `Cvar_SetValue`, `PORT_ANY`, `PORT_SERVER`, `NA_*` address type constants
- `NET_AdrToString` — defined in `qcommon/net_chan.c`, not in this file
- `NET_SendPacket` (higher-level wrapper) — defined in `qcommon/net_chan.c`
