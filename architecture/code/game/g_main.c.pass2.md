# code/game/g_main.c — Enhanced Analysis

## Architectural Role

`g_main.c` is the **VM boundary membrane** between `qcommon`'s VM host and all server-side game logic. It owns the three globally-shared memory regions (`level`, `g_entities`, `g_clients`) that are registered with the server via `trap_LocateGameData`, making them readable by `code/server/sv_snapshot.c` for snapshot generation without any copying. `vmMain` functions as a vtable-in-disguise: because QVM requires one exported symbol, all engine-to-game communication is multiplexed through a single integer-dispatch switch, mirroring the `refexport_t`/`botlib_export_t` vtable pattern used elsewhere but constrained to the bytecode ABI. The cvar registration table (`gameCvarTable`) is the game module's declarative contract with the engine's cvar system, establishing which server-side variables are broadcast to clients (`CVAR_SERVERINFO`), which require map restarts (`CVAR_LATCH`), and which changes should be announced in-game (`trackChange`).

## Key Cross-References

### Incoming (who depends on this file)

- **`code/server/sv_game.c`** calls `VM_Call(gvm, GAME_RUN_FRAME, ...)`, `VM_Call(gvm, GAME_INIT, ...)`, etc. — these land in `vmMain`. The server never calls game functions directly; all traffic passes through the switch.
- **`code/server/sv_snapshot.c`** reads `g_entities` and `g_clients` directly (registered via `trap_LocateGameData`) to build delta-compressed snapshots — the only cross-boundary direct memory access in the entire engine.
- **`code/server/sv_client.c`** drives `GAME_CLIENT_CONNECT`, `GAME_CLIENT_DISCONNECT`, `GAME_CLIENT_BEGIN`, `GAME_CLIENT_COMMAND`, and `GAME_CLIENT_USERINFO_CHANGED` commands through `vmMain` as client lifecycle events occur.
- `level`, `g_entities`, and the game cvars are read by virtually every other file in `code/game/` — they are the de-facto global state store for the whole game module.

### Outgoing (what this file depends on)

- **`code/server/` (via `trap_*` syscalls in `g_syscalls.c`)**: `trap_LocateGameData`, `trap_SetConfigstring`, `trap_SendServerCommand`, `trap_FS_FOpenFile`, `trap_GetServerinfo`, `trap_Cvar_Register/Update/Set` — all cross the VM boundary.
- **`g_client.c` / `g_active.c`**: `ClientConnect`, `ClientDisconnect`, `ClientBegin`, `ClientUserinfoChanged`, `ClientThink`, `ClientEndFrame`, `ClientCommand` — the full client lifecycle is delegated here.
- **`ai_main.c` / `g_bot.c`**: `BotAISetup`, `BotAIShutdown`, `BotAILoadMap`, `BotAIStartFrame`, `BotInterbreedEndMatch` — bot AI is driven from the frame and init/shutdown hooks.
- **`g_spawn.c`**: `G_SpawnEntitiesFromString` — the entire entity world is built here during `G_InitGame`.
- **`g_svcmds.c`**: `ConsoleCommand`, `G_ProcessIPBans` — operator console access.
- **`g_combat.c`, `g_missile.c`, `g_items.c`, `g_mover.c`**: via `G_RunMissile`, `G_RunItem`, `G_RunMover`, `G_RunClient`, `G_RunThink` in the `G_RunFrame` loop.
- **`code/botlib/` (indirectly)**: `BotAIStartFrame` dispatches through `trap_BotLib*` syscalls → `sv_bot.c` → botlib, never linked directly.

## Design Patterns & Rationale

- **Declarative cvar table**: `gameCvarTable` separates cvar *policy* (name, default, flags, announcement behavior) from *mechanism*. Adding a new cvar requires only a table row, not scattered `trap_Cvar_Register` calls. The `trackChange`/`teamShader` boolean fields extend this without subclassing — a pragmatic flat-struct approach over any inheritance model.
- **Single-entry VM dispatch**: The `vmMain` switch pattern is forced by QVM's requirement for one exported function, but also provides a clean audit surface for all engine-to-game calls. In native DLL mode this becomes a simple function call overhead; in QVM mode it is the only legal call gate, enforcing sandboxing.
- **Pre-allocating entity slots for clients**: Setting `level.num_entities = MAX_CLIENTS` before spawning any map entities ensures client entity numbers never alias with world entities, allowing the server to index into `g_entities` by client number without range checks. This is a deliberate reservation, not an oversight.
- **`G_RunThink` hard-errors on NULL think**: Rather than silently skipping entities with elapsed `nextthink` and no `think` pointer, the code calls `G_Error`. This is a defense against stale `nextthink` values left by sloppy entity logic — fail-fast over silent corruption.

## Data Flow Through This File

```
Engine
  │
  ▼  vmMain(GAME_INIT, levelTime, seed, restart)
G_InitGame ──► G_RegisterCvars ──► trap_Cvar_Register (each row in gameCvarTable)
           ──► memset(g_entities, g_clients, level) ──► zeroes all game state
           ──► trap_LocateGameData ──────────────────────► sv_snapshot.c reads directly
           ──► G_SpawnEntitiesFromString ──► g_entities populated
           ──► BotAILoadMap ──► trap_BotLib* ──► sv_bot.c ──► botlib

  ▼  vmMain(GAME_RUN_FRAME, levelTime)
G_RunFrame ──► G_UpdateCvars (poll changes, broadcast if trackChange)
           ──► for each entity: G_RunMissile / G_RunItem / G_RunMover / G_RunThink
           ──► for each client: G_RunClient ──► ClientThink (g_active.c)
           ──► ClientEndFrame (syncs playerState → entityState for snapshot)
           ──► CheckExitRules ──► LogExit ──► BeginIntermission ──► ExitLevel

  ▼  CalculateRanks (called from g_client, g_combat on score change)
     ──► sort level.sortedClients by score
     ──► trap_SetConfigstring(CS_SCORES1/2) ──► clients receive rankings
     ──► CheckExitRules (fraglimit / capturelimit check)
```

## Learning Notes

- **QVM ABI constraint shapes the whole file**: In modern engines (Unity, Unreal, idTech 4+), game modules expose typed function pointers. Q3's QVM forces everything through a single integer-tagged trampoline, producing the verbose 12-argument `vmMain` signature. The arguments are untyped `int`s that pack pointers and flags — a concession to the bytecode's inability to pass structs.
- **`trap_LocateGameData` as zero-copy IPC**: The server and game VM share memory by exchanging raw pointers at startup. This is only safe because both live in the same process address space. In QVM (sandboxed) mode the VM's data segment is offset by `dataMask` but the server still resolves the real host addresses. Modern engines would use message passing or a well-typed shared state API.
- **No ECS, but adjacent ideas**: `g_entities` is a flat array of fat structs (`gentity_t` with ~30 fields), closer to a struct-of-arrays design than a component system. The `think`/`nextthink` callback pattern is an early form of deferred behavior without a true scheduler. The `FL_TEAMSLAVE` / `teamchain` / `teammaster` pattern in `G_FindTeams` is a manual linked-list component applied at spawn time — a precursor to ECS tagging.
- **Warmup and vote state live in `level_locals_t`**: These could be separate subsystems; they're monolithic fields on `level` because Q3's design philosophy was simplicity over modularity. The entire game is essentially one large struct (`level`) mutated by one large frame function.
- **`G_RemapTeamShaders` MissionPack coupling**: The `teamShader` field on cvarTable entries and `G_RemapTeamShaders` are `#ifdef MISSIONPACK` islands — an example of how feature additions were bolted onto the declarative cvar table without disturbing base Q3 behavior.

## Potential Issues

- **`vsprintf` without bounds checking** in `G_Printf`/`G_Error` (fixed-size 1024-byte buffer) is a classic C-era stack overflow vector if format strings produce output exceeding 1024 bytes. Modern code would use `vsnprintf`.
- **Dead timing code in `G_RunFrame`**: `start`/`end` variables measuring frame time are computed but their values are never used or logged, suggesting abandoned profiling instrumentation.
- **`G_FindTeams` is O(n²)**: It iterates all entity pairs for team matching. With `MAX_GENTITIES = 1024` this is negligible at runtime (only called once at map load), but the algorithm does not scale to larger entity counts.
