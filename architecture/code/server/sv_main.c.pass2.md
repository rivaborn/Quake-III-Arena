# code/server/sv_main.c — Enhanced Analysis

## Architectural Role

`sv_main.c` is the **authoritative server's integration spine**: it owns the three globals (`svs`, `sv`, `gvm`) that every other server file (`sv_client.c`, `sv_snapshot.c`, `sv_game.c`, `sv_bot.c`, `sv_init.c`) reads and writes, making it the de facto server scene-graph root. It bridges three subsystem boundaries simultaneously: it receives tick authority from `qcommon/common.c` (`Com_Frame` → `SV_Frame`), drives `code/game` via `VM_Call(gvm, GAME_RUN_FRAME)`, and exchanges UDP datagrams through `qcommon`'s `NET_*`/`Netchan_*` layer. In listen-server mode, `code/client/cl_main.c` calls `SV_Frame` and `SV_Shutdown` directly, meaning this file's frame loop runs inside the client process with no IPC boundary.

## Key Cross-References

### Incoming (who depends on this file)

- **`qcommon/common.c`**: `Com_Frame` calls `SV_Frame(msec)` once per engine tick and `SV_PacketEvent` for every incoming UDP datagram; `com_sv_running` (written by `sv_init.c` but tested here) gates the entire server path
- **`code/client/cl_main.c`**: calls `SV_Frame`, `SV_Shutdown`, and `SV_PacketEvent` directly in listen-server mode — these entry points serve double duty as both dedicated-server and embedded-server APIs
- **All `code/server/*.c` files**: read `svs`, `sv`, `gvm`, and the `sv_*` cvar pointers defined in this file's global scope; there is no encapsulation — the globals are `extern`-linked through `server.h`
- **`code/game/g_syscalls.c`**: calls `SV_SendServerCommand` via the `trap_SendServerCommand` syscall to push reliable commands from the game VM up to clients

### Outgoing (what this file depends on)

- **`qcommon` network layer**: `NET_OutOfBandPrint`, `NET_StringToAdr`, `NET_Sleep`, `Huff_Decompress`, `Netchan_Process` (via `SV_Netchan_Process`) — all packet I/O flows through qcommon primitives
- **`qcommon` VM host**: `VM_Call(gvm, GAME_RUN_FRAME, ...)` in `SV_Frame` is the sole point of game-logic execution; `gvm` handle is created in `sv_init.c` and consumed here
- **`qcommon` command/cvar system**: `Cvar_InfoString`, `Cvar_VariableValue`, `Cbuf_AddText` for map restart; `com_dedicated`, `com_speeds`, `time_game` are read-only globals from common
- **`sv_client.c`**: `SV_DropClient`, `SV_DirectConnect`, `SV_GetChallenge`, `SV_ExecuteClientMessage`, `SV_SendClientMessages`
- **`sv_snapshot.c`**: `SV_SendClientMessages` (indirectly drives snapshot build pipeline)
- **`sv_bot.c`**: `SV_BotFrame` called once per `SV_Frame` iteration before the game VM tick

## Design Patterns & Rationale

**Power-of-2 circular ring buffer** for reliable commands (`MAX_RELIABLE_COMMANDS` must be a power of two; index via `& (MAX_RELIABLE_COMMANDS-1)`). This avoids modulo on every packet and is a classic lock-free idiom. The deliberate `==` rather than `>=` on the overflow check (`reliableSequence - reliableAcknowledge == MAX_RELIABLE_COMMANDS + 1`) prevents `SV_DropClient`'s own broadcast print from triggering a recursive drop — a subtle self-protection guard.

**Stateless OOB / stateful sequenced split**: `SV_PacketEvent` pattern-matches on the four `0xFF` OOB header bytes before any per-client lookup, mirroring the Quake networking model where the same UDP socket carries both connectionless and connection-oriented traffic. The challenge token echoed in `SVC_Status`/`SVC_Info` is an explicit anti-spoofing measure against the "ghost server" injection attack that plagued early 2000s server browsers.

**Fixed-step VM execution with time residual**: `SV_Frame` accumulates `msec` into `sv.timeResidual` and fires `GAME_RUN_FRAME` in `1000/sv_fps` ms increments, decoupling the engine frame rate from the game simulation rate. This is the server equivalent of a fixed-timestep physics loop — the game VM always sees uniform time slices regardless of OS scheduling jitter.

**Lazy DNS resolution** for master servers: `sv_master[i]->modified` gates `NET_StringToAdr` calls, avoiding DNS hitches on every heartbeat. Failed resolutions self-clear the cvar to suppress repeated lookups — a pragmatic 1999-era workaround for Win95 DNS blocking the main thread.

**Notable bug**: In `SV_MasterHeartbeat`, `strstr(":", sv_master[i]->string)` has its arguments reversed — it searches for the master server string *inside the literal `":"` string*, which can never succeed. The port assignment logic is therefore dead code.

## Data Flow Through This File

```
Com_Frame (msec)
  │
  ├─ SV_PacketEvent(from, msg)
  │    ├─ [OOB] → SV_ConnectionlessPacket → SVC_Status / SVC_Info / SV_GetChallenge /
  │    │                                     SV_DirectConnect / SVC_RemoteCommand
  │    └─ [sequenced] → find client_t by IP+qport → SV_Netchan_Process
  │                     → SV_ExecuteClientMessage (sv_client.c)
  │
  └─ SV_Frame(msec)
       ├─ accumulate sv.timeResidual
       ├─ SV_BotFrame(svs.time)              [sv_bot.c → botlib]
       ├─ loop: VM_Call(gvm, GAME_RUN_FRAME) [code/game]
       │         SV_SetConfigstring updates
       ├─ SV_CalcPings → writes playerState_t.ping via SV_GameClientNum
       ├─ SV_CheckTimeouts → SV_DropClient on stale clients
       ├─ SV_SendClientMessages              [sv_snapshot.c: build + send snapshots]
       └─ SV_MasterHeartbeat → NET_OutOfBandPrint to master servers
```

Reliable commands flow upward: `SV_SendServerCommand` → `SV_AddServerCommand` (enqueue in ring) → included in next snapshot packet by `sv_snapshot.c`.

## Learning Notes

- **The `gvm` global is the only handle to game logic**: all server files that need to call game code must route through `VM_Call(gvm, ...)`. There is no vtable or interface object — the VM opcode ABI *is* the interface.
- **`serverStatic_t` vs `server_t` split** is idiomatic to Quake engines: `svs` (static) survives `map` reloads; `sv` (per-map) is cleared on each level load in `sv_init.c`. Modern engines achieve the same with scene/session separation.
- **No ECS**: entities are flat `gentity_t` arrays with function-pointer `think` callbacks — a classic 1990s entity model. The server never directly touches `gentity_t`; it accesses entity data only through `entityState_t` snapshots and `playerState_t` (shared memory layout between server and game VM in DLL mode).
- **The 23-day uptime limit** (`svs.time` 32-bit wrap) reflects the era's assumption that servers would be restarted regularly. The graceful shutdown-and-restart via `Cbuf_AddText("map_restart\n")` is the contemporary idiom for handling this.
- **`QDECL`** on `SV_SendServerCommand` marks it for `__cdecl` calling convention on Windows — necessary because this function is called both from engine C code and potentially from VM-compiled game code across DLL boundaries.

## Potential Issues

- **Reversed `strstr` arguments** in `SV_MasterHeartbeat` (line ~230): port-colon detection is permanently broken; all master server addresses always get `BigShort(PORT_MASTER)` assigned regardless of whether they include a port.
- **Static `lasttime` in `SVC_RemoteCommand`**: not zero-initialized in C89 (though zero by default in C for file-scope), but the unsigned subtraction `time < lasttime + 500` wraps on first call if `Com_Milliseconds()` is small — extremely unlikely but theoretically possible at engine startup.
- **`SV_ReplacePendingServerCommands` is commented out**: the optimization to deduplicate configstring updates was disabled (comment says "ugly"), meaning burst configstring changes generate redundant reliable commands, increasing risk of ring overflow on slow clients.
