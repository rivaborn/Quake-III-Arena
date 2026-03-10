# code/server/sv_init.c — Enhanced Analysis

## Architectural Role

`sv_init.c` is the **server lifecycle orchestrator** — the hub through which all major subsystems are coordinated during startup, map transitions, and shutdown. It sits at the boundary between the persistent engine state (`svs`) and the per-level state (`sv`), managing the exact moment each subsystem is brought online or torn down. It is the only file that sequences `qcommon` (hunk/filesystem/CM), the client layer (CL_*), the game VM (via `VM_Call`), and the snapshot system in the correct order during a map load. Within the server subsystem, `sv_snapshot.c` consumes the baselines and snapshot entity pool allocated here, `sv_client.c` manages the per-client connection state machine that originates here, and `sv_game.c` routes game VM syscall reads/writes of configstrings back into the functions defined here.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/server/sv_game.c`** — `SV_GameSystemCalls` directly dispatches `trap_SetConfigstring` / `trap_GetConfigstring` / `trap_GetUserinfo` / `trap_SetUserinfo` to `SV_SetConfigstring`, `SV_GetConfigstring`, `SV_GetUserinfo`, `SV_SetUserinfo` defined here. Every configstring write by the game VM passes through this file.
- **`code/server/sv_ccmds.c`** — Operator commands (`map`, `devmap`, `spmap`) invoke `SV_SpawnServer`; `SV_Init` registers those commands via `SV_AddOperatorCommands`.
- **`code/qcommon/common.c`** — Calls `SV_Init` once at engine startup and `SV_Shutdown` on clean exit or fatal error.
- **`code/server/sv_snapshot.c`** — Reads `sv.svEntities[].baseline` (written by `SV_CreateBaseline`) and `svs.snapshotEntities` / `svs.numSnapshotEntities` (allocated by `SV_Startup` / `SV_ChangeMaxClients`) every frame for delta-compressed entity transmission.
- **`code/server/sv_client.c`** — Reads `svs.clients` array layout and `sv.configstrings[]` for gamestate construction (`SV_SendClientGameState`).

### Outgoing (what this file depends on)

- **`qcommon` layer** — `Hunk_Alloc/Clear/SetMark`, `Z_Malloc/Free`, `CopyString`, `CM_LoadMap/ClearMap`, `FS_Restart/ClearPakReferences/LoadedPakChecksums`, `VM_Call`, `Cvar_Get/Set`, `Com_Milliseconds/Printf/Error/Memset`.
- **Client layer** — `CL_MapLoading`, `CL_ShutdownAll`, `CL_Disconnect` called from `SV_SpawnServer` / `SV_Shutdown` to synchronize the listen-server client with the server's state transitions.
- **Game VM** — `SV_InitGameProgs`, `SV_ShutdownGameProgs`, `VM_Call(gvm, GAME_RUN_FRAME/GAME_CLIENT_CONNECT/GAME_CLIENT_BEGIN)`.
- **Bot system** — `SV_BotFrame`, `SV_BotInitCvars`, `SV_BotInitBotLib` wired in during `SV_Init` and used during settling frames in `SV_SpawnServer`.
- **World / snapshot** — `SV_ClearWorld`, `SV_SendClientSnapshot` (called in `SV_FinalMessage`), `SV_DropClient`, `SV_Heartbeat_f`, `SV_MasterShutdown`.

## Design Patterns & Rationale

- **Two-tier state structure (`sv` vs `svs`)**: `sv` is wiped by `SV_ClearServer` on every map change; `svs` persists across maps and holds allocations (the client array, snapshot entity pool) that survive level transitions. This clean separation avoids reconnecting clients from scratch on every map change — a critical UX decision for LAN/internet play.
- **Guarded configstring broadcast**: `SV_SetConfigstring` checks `sv.state == SS_GAME || sv.restarting` before broadcasting. This allows `SV_InitGameProgs` to write dozens of configstrings during map load (in `SS_LOADING`) without generating a burst of unreliable network traffic to already-connected clients; they receive the entire gamestate in one block at transition time.
- **Chunked configstring protocol (`bcs0/bcs1/bcs2`)**: Configstrings can theoretically reach several KB (e.g. `CS_SYSTEMINFO` containing all `CVAR_SERVERINFO` vars). The `bcs*` commands allow reliable reassembly on the client (`cl_parse.c`) across multiple MAX_STRING_CHARS-limited messages. This is a workaround for the engine's fixed-size message buffer, not a general streaming design.
- **Hunk mark / temp memory pattern**: After `SV_SpawnServer` finishes, `Hunk_SetMark` is called to record the high-water mark. Subsequent allocations (render assets, bot routing caches) go above this mark and can be freed independently. The temp memory scratch buffer in `SV_ChangeMaxClients` follows the same transient-alloc idiom.
- **Random checksum feed**: `sv.checksumFeed = srand(Com_Milliseconds()) ^ rand()` passed to `FS_Restart` seeds the pak signature verification. Pure server validation (`sv_pure`) uses this feed so each map load generates a unique expected-checksum set, preventing clients from substituting paks between level loads.

## Data Flow Through This File

```
SV_Init ──► registers ~30 cvar_t* globals (sv_maxclients, sv_pure, etc.)

SV_SpawnServer(server, killBots):
  ├── SV_ShutdownGameProgs()          [tears down old gvm]
  ├── CL_MapLoading/CL_ShutdownAll   [client layer sync]
  ├── Hunk_Clear()                    [wipe all per-map allocs]
  ├── CM_ClearMap → CM_LoadMap(bsp)   [collision world rebuilt]
  ├── SV_ClearServer()                [sv.configstrings[] freed + sv memset]
  ├── FS_Restart(checksumFeed)        [vfs repopulated with new pk3 set]
  ├── SV_InitGameProgs()              [gvm created; game calls SV_SetConfigstring
  │                                    many times → sv.configstrings[] filled]
  ├── 4× VM_Call(GAME_RUN_FRAME)      [entities settle; configstrings stabilize]
  ├── SV_CreateBaseline()             [sv.svEntities[].baseline ← svent->s]
  ├── for each client: reconnect      [CS_CONNECTED or CS_ACTIVE for bots]
  └── sv.state = SS_GAME              [unlocks SV_SetConfigstring broadcasts]

SV_SetConfigstring(index, val):
  sv.configstrings[index] ← CopyString(val)       [heap allocation per string]
  if SS_GAME: for each CS_PRIMED+ client:
    len < maxChunkSize → "cs %i \"%s\""            [single reliable command]
    len ≥ maxChunkSize → "bcs0/.../bcs2" chunks   [reassembled by cl_parse.c]

SV_CreateBaseline:
  sv.svEntities[n].baseline ← sv_GentityNum(n)->s  [read by sv_snapshot.c]
```

## Learning Notes

- **Configstring as the server→client key-value store**: Q3 uses the configstring array (up to 1024 slots, indexed by `CS_*` constants) as the primary mechanism for communicating level metadata — models, sounds, player info, map name, game rules — to clients. This is fundamentally different from modern engines that use property-bag replication or ECS component sync; it's a flat, string-indexed shared state with reliable delivery.
- **No ECS**: Entities are flat C structs in a fixed-size array accessed by index (`SV_GentityNum(i)`). The baseline system is the engine's only concession to delta-compression at the entity level; it is set once per map, not updated per-frame (per-frame deltas are handled in `sv_snapshot.c`).
- **VM settling frames**: Running 4 game frames (`3 + 1`) before sending the gamestate to clients is idiomatic to id Tech 3. It lets think functions, trigger volumes, and mover entities reach stable positions so the baseline snapshot is representative. Modern engines accomplish this via a more explicit initialization phase.
- **Bot network address hack**: `netchan.remoteAddress.type == NA_BOT` is the sentinel that identifies a bot client — bots have no real UDP socket, they're driven directly by the game VM. `SV_SpawnServer` uses this to decide whether to fast-path a bot directly to `CS_ACTIVE` (skipping the handshake round trip that would never arrive).
- **`SV_TouchCGame`**: Pure-server enforcement requires that the server "reference" every pk3 that clients must have. Touching `vm/cgame.qvm` via `FS_FOpenFileRead` causes the filesystem to mark that pak as referenced, so `FS_ReferencedPakChecksums` includes it in the pure validation list sent in `CS_SYSTEMINFO`.

## Potential Issues

- **`SV_ChangeMaxClients` memory hazard**: The function copies only slots `[0..count-1]` (where `count` is the highest active slot index + 1). If `sv_maxclients` is lowered below `count`, `SV_BoundMaxClients(count)` prevents the shrink — but the logic iterating `sv_maxclients->integer` in `SV_SetConfigstring` and snapshot code reads the new value, not the old one, creating a possible window if the cvar is modified mid-frame before `SV_ChangeMaxClients` runs.
- **Configstring memory leak on error**: `SV_SetConfigstring` calls `Z_Free` then `CopyString`; if `CopyString` fatally errors (OOM), the old pointer is already freed. In practice Q3's zone allocator calls `Com_Error(ERR_FATAL)` on OOM, which unwinds the stack via `longjmp`, so this is unlikely to manifest as a silent leak but could leave `sv.configstrings[index]` as a dangling pointer on the error recovery path.
- **`SV_SpawnServer` not safe for map_restart**: The comment is explicit, but there is no guard — calling `SV_SpawnServer` when `sv.restarting` is true would double-clear already-cleared state. The restart path (`SV_RestartGameProgs` in `sv_game.c`) is a separate, non-overlapping code path, so this is a documentation hazard rather than a runtime one.
