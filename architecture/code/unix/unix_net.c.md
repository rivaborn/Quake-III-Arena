# code/unix/unix_net.c

## File Purpose
Implements the Unix/Linux (and macOS) platform-specific network layer for Quake III Arena, providing UDP socket creation, packet send/receive, local address enumeration, and LAN classification. It fulfills the `Sys_*` and `NET_*` network API required by the engine's platform-agnostic common layer (`qcommon`).

## Core Responsibilities
- Convert between engine `netadr_t` and POSIX `sockaddr_in` representations
- Open and close UDP sockets for IP (and stub IPX) communication
- Send and receive raw UDP packets
- Enumerate the host's local IP addresses (platform-divergent: Mac vs. generic POSIX)
- Classify an address as LAN or WAN (RFC 1918 class A/B/C awareness)
- Provide a blocking/sleeping select-based idle for dedicated server frame throttling

## Key Types / Data Structures
None declared locally; relies on types from included headers.

| Name | Kind | Purpose |
|------|------|---------|
| `netadr_t` | typedef struct (extern, `qcommon.h`) | Engine network address (type + IPv4 bytes + port) |
| `msg_t` | typedef struct (extern, `qcommon.h`) | Message buffer with read/write cursor used for packet data |
| `cvar_t` | typedef struct (extern, `q_shared.h`) | Console variable handle |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `noudp` | `cvar_t *` | static (file) | Cvar `net_noudp`; if non-zero, disables UDP initialization |
| `net_local_adr` | `netadr_t` | global | Engine's local network address (declared but not populated here) |
| `ip_socket` | `int` | global | File descriptor for the active UDP/IP socket |
| `ipx_socket` | `int` | global | File descriptor for IPX socket (always 0/unused on Unix) |
| `numIP` | `int` | static (file) | Count of discovered local IPs |
| `localIP` | `byte[MAX_IPS][4]` | static (file) | Array of discovered local IPv4 addresses (up to 16) |

## Key Functions / Methods

### NetadrToSockadr
- **Signature:** `void NetadrToSockadr(netadr_t *a, struct sockaddr_in *s)`
- **Purpose:** Converts an engine `netadr_t` to a POSIX `sockaddr_in` for use in socket calls.
- **Inputs:** Engine address `a` (NA_BROADCAST sets broadcast IP; NA_IP copies raw bytes).
- **Outputs/Return:** Populates `*s` in-place.
- **Side effects:** None.
- **Calls:** `memset`
- **Notes:** Only handles NA_BROADCAST and NA_IP; other types leave `*s` zeroed.

### SockadrToNetadr
- **Signature:** `void SockadrToNetadr(struct sockaddr_in *s, netadr_t *a)`
- **Purpose:** Converts a POSIX `sockaddr_in` back to an engine `netadr_t`; always sets type to NA_IP.
- **Inputs:** `*s` from a recvfrom call.
- **Outputs/Return:** Populates `*a` in-place.
- **Side effects:** None.
- **Calls:** None.

### Sys_StringToSockaddr
- **Signature:** `qboolean Sys_StringToSockaddr(const char *s, struct sockaddr *sadr)`
- **Purpose:** Resolves a hostname or dotted-decimal IP string to a `sockaddr`.
- **Inputs:** String `s`; may be numeric or a DNS name.
- **Outputs/Return:** `qtrue` on success, `qfalse` if DNS resolution fails.
- **Side effects:** Blocking DNS lookup via `gethostbyname` if non-numeric.
- **Calls:** `memset`, `inet_addr`, `gethostbyname`

### Sys_GetPacket
- **Signature:** `qboolean Sys_GetPacket(netadr_t *net_from, msg_t *net_message)`
- **Purpose:** Drains one packet from the IP or IPX socket into `net_message`.
- **Inputs:** Pointers to destination address and message buffer.
- **Outputs/Return:** `qtrue` if a valid packet was read; `qfalse` if no data or error.
- **Side effects:** Modifies `net_message->data`, `cursize`, `readcount`; prints on non-EWOULDBLOCK errors.
- **Calls:** `recvfrom`, `SockadrToNetadr`, `Com_Printf`, `NET_ErrorString`, `NET_AdrToString`
- **Notes:** Iterates over both ip_socket and ipx_socket; skips EWOULDBLOCK and ECONNREFUSED silently; drops oversize packets with a warning.

### Sys_SendPacket
- **Signature:** `void Sys_SendPacket(int length, const void *data, netadr_t to)`
- **Purpose:** Sends a UDP datagram to the given engine address.
- **Inputs:** Byte count, data pointer, destination `netadr_t`.
- **Outputs/Return:** None.
- **Side effects:** Calls `sendto`; prints on error; calls `Com_Error(ERR_FATAL)` for unknown address types.
- **Calls:** `NetadrToSockadr`, `sendto`, `Com_Error`, `Com_Printf`, `NET_ErrorString`, `NET_AdrToString`

### Sys_IsLANAddress
- **Signature:** `qboolean Sys_IsLANAddress(netadr_t adr)`
- **Purpose:** Returns true if the address is loopback, IPX, or shares an IP subnet with a local interface (RFC 1918 class A/B/C matching).
- **Inputs:** Engine address to test.
- **Outputs/Return:** `qtrue`/`qfalse`.
- **Side effects:** None.
- **Calls:** None (pure logic against `localIP` table).
- **Notes:** Class B check also recognises 172.16.0.0/12; Class C check also recognises 192.168.0.0/16.

### NET_GetLocalAddress
- **Signature:** `void NET_GetLocalAddress(void)`
- **Purpose:** Populates `localIP[]` and `numIP` with all non-loopback AF_INET interface addresses.
- **Inputs:** None.
- **Outputs/Return:** None (side effects only).
- **Side effects:** On macOS: opens/closes a temporary DGRAM socket; uses `SIOCGIFCONF`/`OSIOCGIFADDR` ioctls. On generic POSIX: calls `gethostbyname` with the machine hostname. Prints discovered IPs to console.
- **Calls:** `gethostname`/`gethostbyname` (POSIX) or `socket`, `ioctl`, `close` (macOS); `Com_Printf`
- **Notes:** macOS path iterates AF_LINK + AF_INET pairs to skip loopback. POSIX path has an off-by-one bug: stores to `localIP[numIP]` after `numIP++`.

### NET_IPSocket
- **Signature:** `int NET_IPSocket(char *net_interface, int port)`
- **Purpose:** Creates, configures (non-blocking, broadcast), and binds a UDP socket.
- **Inputs:** Interface name or NULL (bind to INADDR_ANY); port number.
- **Outputs/Return:** Valid socket fd on success, 0 on failure.
- **Side effects:** Allocates a kernel socket; prints errors to console.
- **Calls:** `socket`, `ioctl (FIONBIO)`, `setsockopt (SO_BROADCAST)`, `Sys_StringToSockaddr`, `htons`, `bind`, `close`, `Com_Printf`, `NET_ErrorString`

### NET_Init / NET_OpenIP / NET_Shutdown
- **Notes:** `NET_Init` reads `net_noudp` cvar and calls `NET_OpenIP`. `NET_OpenIP` tries up to 10 consecutive ports and calls `NET_GetLocalAddress` on success; fatal error if all fail. `NET_Shutdown` closes `ip_socket`.

### NET_Sleep
- **Signature:** `void NET_Sleep(int msec)`
- **Purpose:** Blocks the dedicated server thread up to `msec` milliseconds or until the IP socket or stdin has data.
- **Inputs:** Timeout in milliseconds.
- **Side effects:** Calls `select()`; reads `stdin_active` extern.
- **Notes:** No-ops if not a dedicated server (`com_dedicated->integer == 0`) or if no socket is open.

## Control Flow Notes
- **Init:** `NET_Init` → `NET_OpenIP` → `NET_IPSocket` + `NET_GetLocalAddress`. Called once during engine startup.
- **Frame:** `Sys_GetPacket` is polled by the common layer each frame to consume incoming UDP datagrams. `Sys_SendPacket` is called on demand to transmit.
- **Shutdown:** `NET_Shutdown` closes sockets. `NET_Sleep` is used by the dedicated server main loop to yield CPU between frames.

## External Dependencies
- `../game/q_shared.h` — `qboolean`, `byte`, `netadr_t`, `cvar_t`, `Com_Printf`, `Com_Error`, `Q_stricmp`, `Com_sprintf`
- `../qcommon/qcommon.h` — `msg_t`, `netadrtype_t`, `NET_AdrToString`, `Cvar_Get`, `Cvar_SetValue`, `PORT_SERVER`, `com_dedicated`
- POSIX headers: `<sys/socket.h>`, `<netinet/in.h>`, `<netdb.h>`, `<arpa/inet.h>`, `<sys/ioctl.h>`, `<errno.h>`
- macOS-only: `<sys/sockio.h>`, `<net/if.h>`, `<net/if_dl.h>`, `<net/if_types.h>`
- **Defined elsewhere:** `NET_AdrToString`, `com_dedicated`, `stdin_active`
