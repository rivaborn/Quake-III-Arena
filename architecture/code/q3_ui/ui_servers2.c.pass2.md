# code/q3_ui/ui_servers2.c — Enhanced Analysis

## Architectural Role

This file implements a **dynamic server discovery and connection UI**, occupying the bridge between user intent (selecting a server) and engine network commands. As part of the UI VM (`code/q3_ui`), it runs in a sandboxed QVM and communicates solely through the `trap_*` syscall ABI defined in `ui_public.h`. It is one of the most complex UI menus in the engine, managing a real-time async ping cycle while responding to user filter/sort interactions—a pattern idiomatic to Q3A's "live" server browser paradigm.

## Key Cross-References

### Incoming (who depends on this file)
- **UI framework** (`code/q3_ui/ui_main.c`, `ui_atoms.c`): Menu registration via `UI_ArenaServersMenu()` entry point; menu stack (`UI_PushMenu`/`UI_PopMenu`) management
- **User input dispatch**: Input events (`K_MOUSE1`, `K_ESCAPE`, space bar) routed to `ArenaServers_MenuDraw` and `ArenaServers_Event` via the menu framework's per-frame event handler
- **Renderer VM syscalls**: All 2D drawing (`trap_R_DrawStretchPic`, `trap_R_SetColor`, etc.) delegated to the renderer back-end running in the main engine process

### Outgoing (what this file depends on)
- **LAN browser syscalls** (`trap_LAN_GetServerCount`, `trap_LAN_GetServerAddressString`, `trap_LAN_GetPing`, `trap_LAN_GetPingInfo`, `trap_LAN_ClearPing`, `trap_LAN_GetPingQueueCount`): Implemented in `code/client/cl_main.c` and `code/qcommon/net_chan.c`; these are the **only** syscalls that access the engine's internal server list and UDP ping queue
- **Command execution** (`trap_Cmd_ExecuteText`): Routes `localservers`, `globalservers`, `getservers`, `connect`, and `ping` commands to `code/qcommon/cmd.c` for execution in the main engine loop
- **Cvar access** (`trap_Cvar_VariableValue`, `trap_Cvar_Set`, `trap_Cvar_Update`): Read/write `cl_maxPing`, `net_masterServer`, and `serverX` (for favorites) via `code/qcommon/cvar.c`
- **Renderer syscalls** (drawing): Shader registration, color setting, image stretching delegate to `code/renderer/tr_image.c`, `tr_main.c`

## Design Patterns & Rationale

### 1. **Async Ping Queue with Frame-Gated Dispatch**
The ping cycle (`ArenaServers_DoRefresh`) runs **per-frame** at ~10 Hz, dispatching only 1–2 new ping requests per frame even though the engine's UDP ping queue (`code/qcommon/net_chan.c`) can buffer `MAX_PINGREQUESTS` (32). This throttling prevents:
- **Packet storm**: Sending all pings simultaneously would spike network load
- **UI stall**: Waiting synchronously for responses would freeze the menu
- **Server overload**: Respecting a human-perceptible refresh rate is friendlier to master servers

### 2. **Monolithic Menu State Structure**
The `arenaservers_t` struct bundles widgets, server lists, ping queues, and filter state into one global. This is **idiomatic to Q3A's pre-widget-tree UI** (pre-MissionPack). Modern engines would decompose this into separate controller/model/view layers; Q3A's design favors simplicity and tight coupling for performance.

### 3. **Cvar-Based Persistence Without Manual Serialization**
Favorite server addresses (`server1`–`server16`) are stored as **cvars**, not in a file. This reuses Q3A's existing `cvar.c` infrastructure for:
- Automatic loading from `q3config.cfg` on startup
- Automatic saving to `q3config.cfg` on shutdown (if `CVAR_ARCHIVE` is set)
- Network transmission to dedicated servers (if `CVAR_USERINFO` is set)

This avoids reimplementing file I/O or serialization logic.

### 4. **Callback-Driven Event Routing**
The `ArenaServers_Event` callback routes **all** user interactions (dropdown changes, button clicks, list selection) to a single central point. This centralization simplifies state management but obscures event flow compared to event-sourcing patterns in modern UI frameworks.

### 5. **Max-Ping Culling vs. Soft Filtering**
Servers with ping ≥ `cl_maxPing` are **silently dropped during insert** (`ArenaServers_Insert`), except for favorites. This is a hard filter, not a soft one. Rationale:
- Favorites always display, even if stale, to avoid losing user bookmarks
- For active queries, servers exceeding the ping threshold won't appear, so users see only "playable" servers

## Data Flow Through This File

```
INIT PHASE:
  ArenaServers_MenuInit
    → ArenaServers_SetType(AS_LOCAL)
      → ArenaServers_StartRefresh
        → trap_Cmd_ExecuteText("localservers ...")   [seeds the engine's ping queue]
        → refreshservers = qtrue

PING CYCLE (per-frame while refreshservers == true):
  ArenaServers_MenuDraw
    → ArenaServers_DoRefresh
      → trap_LAN_GetPingQueueCount()  [check how many pings are pending]
      → for each completed ping:
          → trap_LAN_GetPingInfo(...)  [harvest result]
          → ArenaServers_Insert(...)   [parse into servernode_t, insert into *serverlist]
      → if queue < MAX_PINGREQUESTS and more servers to ping:
          → trap_Cmd_ExecuteText("ping <adr>")  [dispatch next ping]
      → if all servers pinged:
          → ArenaServers_StopRefresh()
            → qsort(*serverlist)  [final sort by g_sortkey]
            → ArenaServers_UpdateMenu()  [rebuild display strings with filters]

FILTER/SORT CHANGES:
  ArenaServers_Event (dropdown / radio button activation)
    → Update g_sortkey / g_gametype / g_emptyservers / g_fullservers
    → qsort(*serverlist)
    → ArenaServers_UpdateMenu()  [re-render with new filters/sort]

CONNECT:
  ArenaServers_Event (ID_CONNECT button)
    → ArenaServers_Go
      → trap_Cmd_ExecuteText("connect <adr>")  [routed to code/client/cl_main.c]

SHUTDOWN:
  ArenaServers_MenuDraw → Menu_DefaultKey(K_ESCAPE)
    → ArenaServers_StopRefresh()
    → ArenaServers_SaveChanges()  [writes favorite cvars]
    → UI_PopMenu()
```

## Learning Notes

### Idiomatic Q3A Patterns
1. **Syscall barrier design**: Every engine-side resource (server lists, ping results, cvars, shaders) is accessed **only** through a typed syscall function. This enforces:
   - Sandbox safety: UI VM cannot corrupt engine memory
   - Version stability: Syscall ABIs are version-negotiated; old UIs work with new engines if the ABI is preserved
   - Platform independence: Syscalls work identically on x86, PPC, and even QVM interpreters

2. **Frame-synchronized async I/O**: The ping queue is asynchronous (UDP responses arrive whenever), but the UI **polls** it once per frame. This avoids callback complexity and keeps the frame loop synchronous and deterministic.

3. **Favorites as cvars**: Storing favorites in cvars rather than a dedicated file reuses the config serialization layer. Modern engines would use JSON or a database, but Q3A's model is simpler and tightly integrated with the cvar system.

### Modern Engine Contrasts
- **No observer pattern**: Filter/sort changes don't emit events; instead, the UI directly re-renders the entire list. This is inefficient for large lists (1000+ servers) but fast enough for Q3A's typical 100–200 visible servers.
- **No entity component system**: Server data is a flat array of `servernode_t`, not a component-based model. Iteration and filtering are O(n) linear scans, not spatial-indexed or cached queries.
- **Blocking syscalls disguised as async**: The `trap_LAN_*` syscalls appear asynchronous from the UI's perspective, but they are **synchronous** on the engine side; the engine's `Com_Frame` loop simply caches results in the LAN server list between calls.

## Potential Issues

1. **Buffer overflow risk in `ArenaServers_Insert`**: The `info` string parsing (hostname, mapname, gamename) uses `Q_strncpy` with fixed buffer sizes (e.g., `MAX_HOSTNAMELENGTH+3`). Malformed server info strings from rogue servers could overflow.

2. **Memory corruption on list overflow**: When `*g_arenaservers.numservers >= maxservers`, the code **overwrites the last slot** rather than rejecting new servers. This could silently truncate the list or cause thrashing if the engine queries more servers than `MAX_GLOBALSERVERS` (128) can hold.

3. **No timeout on ping requests**: If a server never responds, its ping slot in `g_arenaservers.pinglist[]` occupies a slot indefinitely. The code assumes the engine clears stale pings via `trap_LAN_ClearPing`, but there's no explicit timeout.

4. **Unchecked cvar writes**: `ArenaServers_SaveChanges` blindly writes `server1`–`server16` cvars without checking if they exceed the engine's `MAX_CVARS` limit or cvar name length. A favorite with a malformed address could corrupt the cvar namespace.
