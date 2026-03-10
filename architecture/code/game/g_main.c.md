# code/game/g_main.c

## File Purpose
The central game module entry point for Quake III Arena's server-side game logic. It owns the VM dispatch table (`vmMain`), manages game initialization/shutdown, drives the per-frame update loop, and maintains all game-wide cvars and level state.

## Core Responsibilities
- Expose `vmMain` as the sole entry point from the engine into the game VM
- Register and update all server-side cvars via `gameCvarTable`
- Initialize and tear down the game world (`G_InitGame`, `G_ShutdownGame`)
- Drive the per-frame entity update loop (`G_RunFrame`)
- Manage tournament warmup, voting, team voting, and exit rules
- Compute and broadcast player/team score rankings
- Handle level intermission sequencing and map transitions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cvarTable_t` | struct | Maps a `vmCvar_t*` to its name, default, flags, and change-tracking metadata |
| `level_locals_t` | struct (defined in g_local.h) | All transient per-level state: clients, entities, scores, voting, intermission |
| `gentity_t` | struct (defined in g_local.h) | All-purpose game entity (players, items, movers, missiles, etc.) |
| `gclient_t` | struct (defined in g_local.h) | Per-client game state, persists across respawns |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `level` | `level_locals_t` | global | Master per-level state cleared on each map load |
| `g_entities` | `gentity_t[MAX_GENTITIES]` | global | Flat entity array shared with the server |
| `g_clients` | `gclient_t[MAX_CLIENTS]` | global | Flat client array shared with the server |
| `gameCvarTable` | `cvarTable_t[]` | static | Declarative table of all game cvars |
| `gameCvarTableSize` | `int` | static | Element count of `gameCvarTable` |
| All `g_*` / `pmove_*` `vmCvar_t` | `vmCvar_t` | global | Individual mirrored cvar handles (~50 total) |

## Key Functions / Methods

### vmMain
- **Signature:** `int vmMain(int command, int arg0..arg11)`
- **Purpose:** Engine-to-game dispatch; the only legal entry into the game module.
- **Inputs:** `command` selects the operation (GAME_INIT, GAME_RUN_FRAME, GAME_CLIENT_CONNECT, etc.); `arg0–arg11` carry command-specific parameters.
- **Outputs/Return:** 0 on success, cast pointer for GAME_CLIENT_CONNECT, -1 for unknown command.
- **Side effects:** Delegates to all major game subsystems.
- **Calls:** `G_InitGame`, `G_ShutdownGame`, `ClientConnect`, `ClientThink`, `ClientUserinfoChanged`, `ClientDisconnect`, `ClientBegin`, `ClientCommand`, `G_RunFrame`, `ConsoleCommand`, `BotAIStartFrame`.
- **Notes:** Must be the first compiled symbol in the .q3vm; command dispatch is a plain switch.

### G_InitGame
- **Signature:** `void G_InitGame(int levelTime, int randomSeed, int restart)`
- **Purpose:** Full game-world initialization for a new map load or restart.
- **Inputs:** Starting time, RNG seed, restart flag.
- **Outputs/Return:** void.
- **Side effects:** Zeroes `level` and both arrays; opens the log file; calls `trap_LocateGameData` to register entity/client pointers with the server; spawns all entities; initialises bots.
- **Calls:** `G_RegisterCvars`, `G_ProcessIPBans`, `G_InitMemory`, `G_InitWorldSession`, `G_SpawnEntitiesFromString`, `G_FindTeams`, `G_CheckTeamItems`, `BotAISetup`, `BotAILoadMap`, `G_InitBots`, `G_RemapTeamShaders`.
- **Notes:** `level.num_entities` is pre-set to `MAX_CLIENTS` so client slots are never reused for other entities.

### G_ShutdownGame
- **Signature:** `void G_ShutdownGame(int restart)`
- **Purpose:** Flush log, persist session data, shut down bot AI.
- **Calls:** `G_LogPrintf`, `G_WriteSessionData`, `BotAIShutdown`.

### G_RunFrame
- **Signature:** `void G_RunFrame(int levelTime)`
- **Purpose:** Per-frame update: advance time, run all entity think functions, perform end-of-frame client fixup, check rules.
- **Inputs:** Current server time in milliseconds.
- **Side effects:** Mutates `level.time`, `level.framenum`; calls think callbacks on all active entities; clears stale events; frees temp entities.
- **Calls:** `G_UpdateCvars`, `G_RunMissile`, `G_RunItem`, `G_RunMover`, `G_RunClient`, `G_RunThink`, `ClientEndFrame`, `CheckTournament`, `CheckExitRules`, `CheckTeamStatus`, `CheckVote`, `CheckTeamVote` (×2), `CheckCvars`.
- **Notes:** Returns immediately if `level.restarted` is set; `start`/`end` timing variables are present but their values are unused (dead debug code).

### G_RegisterCvars / G_UpdateCvars
- **Purpose:** Register all cvars from `gameCvarTable` at init; poll for changes every frame and broadcast announced changes to all clients.
- **Side effects:** Calls `G_RemapTeamShaders` if any `teamShader` cvar changed.

### CalculateRanks
- **Signature:** `void CalculateRanks(void)`
- **Purpose:** Recount connected/playing/voting clients, sort by score, assign `PERS_RANK`, update CS_SCORES configstrings, check exit rules.
- **Side effects:** Modifies `level.sortedClients`, `level.numConnectedClients`, `level.numPlayingClients`, `level.numVotingClients`; calls `trap_SetConfigstring`, `CheckExitRules`, `SendScoreboardMessageToAllClients`.
- **Notes:** Called on every connect, disconnect, death, and team change.

### CheckExitRules
- **Purpose:** Evaluate timelimit, fraglimit, and capturelimit each frame; queue intermission via `LogExit` when a limit is hit; handles sudden-death tie detection.

### BeginIntermission / ExitLevel
- **Purpose:** `BeginIntermission` freezes all clients at the intermission point and sends scores. `ExitLevel` transitions to the next map (or restarts tournament).

### G_FindTeams
- **Purpose:** Link all entities sharing a `team` string into a master/slave chain; canonicalizes `targetname` onto the master.

### G_RunThink
- **Signature:** `void G_RunThink(gentity_t *ent)`
- **Purpose:** Fire `ent->think` if `nextthink` has elapsed.
- **Notes:** Errors (not silently skips) if `think` is NULL when the time arrives.

## Control Flow Notes
- **Init:** `vmMain(GAME_INIT)` → `G_InitGame` (one-shot at map load).
- **Frame:** `vmMain(GAME_RUN_FRAME)` → `G_RunFrame` (called once per server frame, ~100 ms).
- **Client events:** `vmMain(GAME_CLIENT_*)` dispatched as they arrive between frames.
- **Shutdown:** `vmMain(GAME_SHUTDOWN)` → `G_ShutdownGame`.

## External Dependencies
- `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`) — all shared types and trap declarations
- `trap_*` syscalls — defined in the engine, bridged through `g_syscalls.c`; cover FS, cvars, server commands, entity linking, AAS, bot lib, etc.
- `ClientConnect`, `ClientThink`, `ClientBegin`, `ClientDisconnect`, `ClientCommand`, `ClientUserinfoChanged`, `ClientEndFrame` — defined in `g_client.c` / `g_active.c`
- `BotAISetup`, `BotAIShutdown`, `BotAILoadMap`, `BotAIStartFrame`, `BotInterbreedEndMatch` — defined in `ai_main.c` / `g_bot.c`
- `G_SpawnEntitiesFromString`, `G_CheckTeamItems`, `UpdateTournamentInfo`, `SpawnModelsOnVictoryPads`, `CheckTeamStatus` — defined elsewhere in the game module
