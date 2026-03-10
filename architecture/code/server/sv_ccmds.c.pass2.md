# code/server/sv_ccmds.c — Enhanced Analysis

## Architectural Role

`sv_ccmds.c` implements the operator/admin command layer at the console-to-server boundary. It sits outside the per-frame game loop and server physics simulation, serving as a thin command-dispatch wrapper that bridges the generic console system (`qcommon/cmd.c`) with server lifecycle and client management APIs. Commands registered here are invoked on-demand (not frame-synchronized) and drive synchronous state transitions: map loads, restarts, client drops, and ban notifications. This decouples admin control from game ticks, allowing out-of-band server reconfiguration.

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon/cmd.c**: `Cmd_ExecuteString` dispatches to registered command handlers registered by `SV_AddOperatorCommands` during server init (`sv_main.c`). Console input or remote RCON triggers invocation.
- **sv_main.c**: Calls `SV_AddOperatorCommands()` at startup; sole init-time dependency.

### Outgoing (what this file depends on)
- **sv_main.c**: Heavy dependence on server lifecycle APIs: `SV_SpawnServer` (full level load), `SV_DropClient` (remove client), `SV_ClientEnterWorld` (reconnect), `SV_RestartGameProgs` (reset game VM), `SV_AddServerCommand` (queue server→client message), `SV_SendServerCommand` (broadcast to all or specific clients), `SV_SetConfigstring` (sync warmup state), `SV_GameClientNum` (entity queries).
- **sv_world.c**: `SV_SectorList_f` (world spatial query debugging).
- **qcommon/vm.c**: `VM_Call` and `VM_ExplicitArgPtr` for direct game VM invocation during `SV_MapRestart_f` (GAME_RUN_FRAME, GAME_CLIENT_CONNECT syscalls).
- **qcommon/net_chan.c** / **qcommon/net_*.c**: `NET_StringToAdr` (resolve authorize server by name), `NET_OutOfBandPrint` (OOB UDP packet to ban server), `NET_AdrToString` (format IP).
- **qcommon/cvar.c**: All cvar getters/setters (`Cvar_Get`, `Cvar_Set`, `Cvar_SetValue`, `Cvar_SetLatched`, `Cvar_VariableString`, `Cvar_VariableValue`).
- **qcommon/fs_*.c**: `FS_ReadFile` (validate map BSP exists).
- **qcommon/cmd.c**: `Cmd_Argc`, `Cmd_Argv` (parse console args).

## Design Patterns & Rationale

**Command Handler Idiom**: Each function follows the no-argument console callback signature and reads arguments via `Cmd_Argc`/`Cmd_Argv`. This is the idiomatic Quake III pattern, enabling the engine to register handler functions without binding signatures — the string-based dispatch is generic across all subsystems.

**Dual Variants for Flexibility**: Kick/Ban come in `-name` and `-num` pairs (`SV_Kick_f`/`SV_KickNum_f`, `SV_Ban_f`/`SV_BanNum_f`). Names are user-friendly for console; slots enable scripts and remote tools (RCON) to target clients without string parsing overhead.

**Helper Extraction**: `SV_GetPlayerByName()` and `SV_GetPlayerByNum()` centralize player-lookup logic and fallback handling. This DRYs the kick/ban implementations, which otherwise would duplicate filtering, validation, and error reporting.

**Async Authorization**: `SV_Ban_f` does not immediately drop the client; it only notifies the external authorize server via OOB UDP. This decouples admin-server ban lists from in-game state, allowing a single authority to enforce bans across multiple servers. The pattern mirrors distributed system design: eventual consistency over immediate local enforcement.

**LOOPBACK Protection**: The host player (LOOPBACK address) is protected from kick/ban across four command handlers. This is applied ad-hoc at each call site rather than factored into helper functions, suggesting it was added reactively to prevent admin mistakes.

## Data Flow Through This File

**Map Load** (`SV_Map_f`): Command input → parse map name and command variant (`map`, `devmap`, `spmap`, `spdevmap`) → validate BSP file exists on disk → set gametype and cheat cvars conditionally → call `SV_SpawnServer` (which reinitializes entire server state, reloads game VM, disconnects all clients). The map name is saved locally because `SV_SpawnServer` triggers a config reload that invalidates `Cmd_Argv` state.

**Map Restart** (`SV_MapRestart_f`): Command input → parse optional delay argument → if delay > 0 and no warmup, set `sv.restartTime` and return (scheduling a future restart) → else: toggle `snapFlagServerBit` (signal clients that map restarted), increment `sv.serverId`, reset game VM state inline, run 3 settling frames, reconnect all clients using `GAME_CLIENT_CONNECT` VM call (no full gamestate transmission, reducing latency). This optimization allows fast respawns with fair start times across heterogeneous client load speeds.

**Client Kick/Ban**: Resolve client by name (with fallback color-strip comparison) or slot number → check state and LOOPBACK protection → for kick: call `SV_DropClient` (removes from simulation) → for ban: additionally `NET_OutOfBandPrint` OOB UDP packet to authorize server with client's IP.

**Chat Broadcast** (`SV_Say_f`): Format message with prefix → call `SV_SendServerCommand(NULL, ...)` which queues the message for transmission to all connected clients.

## Learning Notes

**Thin Engine, Thick Game**: The engine provides minimal admin scaffolding — only map loading, client dropping, and messaging. All game rules (CTF flags, respawn logic, team management) live in the game VM. This boundary is visible here: admin commands change *server state*, but the game VM owns the response.

**Warmup Delay Innovation**: The `SV_MapRestart_f` mechanism (delay + in-place reconnect) avoids a full level reload, allowing clients on slow connections to join before the game starts. Modern engines use client-side prediction to hide latency; Q3 engineered the server to stall and buffer clients before frame 0.

**Multi-Variant Command Registration**: The engine registers separate console commands for `map`, `devmap`, `spmap`, `spdevmap` variants, all pointing to `SV_Map_f`. This is the pre-configuration-file era of command systems; modern engines use parameterized CLIs or config-driven registration.

**Distributed IP-Based Bans**: The authorize server integration (`svs.authorizeAddress`) centralizes ban lists across multiple Q3 servers without requiring a login system. IP bans are crude (LAN spoofable, ISP-shared) but fit the 2005 context and are decentralized — servers contact a known authority, but no persistent user account is required.

**VM Invocation Inside Command Handlers**: `SV_MapRestart_f` directly calls `VM_Call(gvm, GAME_RUN_FRAME, ...)` and `VM_ExplicitArgPtr(gvm, VM_Call(...))`. This is a rare instance of off-frame VM invocation; normally the server loop drives frames. It trades frame-synchronization for imperative restart control.

## Potential Issues

- **Dead Code**: `SV_GetPlayerByNum` ends with an unreachable `return NULL` after the first `return cl`. Minor but indicates incomplete editing.
- **Code Duplication**: `SV_Ban_f` and `SV_BanNum_f` are nearly identical except for player lookup. The authorize server logic (DNS resolution, OOB packet) is duplicated verbatim.
- **Ad-Hoc Loopback Protection**: LOOPBACK checks appear in four places rather than factored into `SV_GetPlayerByName`/`SV_GetPlayerByNum`. Fragile to future additions.
- **Inline State Machine in `SV_MapRestart_f`**: Direct manipulation of `sv.state`, `sv.restarting`, VM calls, and client reconnection all inline makes this function long and fragile to interruption or error recovery.
