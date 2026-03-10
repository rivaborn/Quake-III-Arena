# code/game/g_local.h

## File Purpose
Central private header for the Quake III Arena server-side game module (game DLL/VM). It defines all major game-side data structures, declares every cross-file function, enumerates game cvars, and lists the full `trap_*` syscall interface that bridges the game VM to the engine.

## Core Responsibilities
- Define `gentity_t` (the universal server-side entity) and `gclient_t` (per-client runtime state)
- Define `level_locals_t`, the singleton that holds all per-map game state
- Define `clientPersistant_t` and `clientSession_t` for data surviving respawns/levels
- Declare every public function exported between game `.c` files
- Declare all `vmCvar_t` globals used by the game module
- Declare all `trap_*` engine syscall wrappers (filesystem, collision, bot AI, etc.)
- Define entity flag bits (`FL_*`), damage flags (`DAMAGE_*`), and timing constants

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `gentity_t` (`gentity_s`) | struct | Universal server entity; first two fields (`s`, `r`) must match `sharedEntity_t` layout expected by the server |
| `gclient_t` (`gclient_s`) | struct | Per-client runtime data; first field `ps` (`playerState_t`) must be first per server contract |
| `level_locals_t` | struct | Singleton holding all per-level state: entity arrays, time, scores, voting, spawn vars, intermission, body queue |
| `clientPersistant_t` | struct | Client data persisting across respawns; cleared on level/team change |
| `clientSession_t` | struct | Client data persisting across levels via cvar serialization; team, spectator state, win/loss |
| `playerTeamState_t` | struct | CTF/teamplay per-client stats (captures, assists, flag timers) |
| `moverState_t` | enum | Four-state FSM for movers: `MOVER_POS1`, `MOVER_POS2`, `MOVER_1TO2`, `MOVER_2TO1` |
| `clientConnected_t` | enum | `CON_DISCONNECTED` / `CON_CONNECTING` / `CON_CONNECTED` |
| `spectatorState_t` | enum | `SPECTATOR_NOT` / `SPECTATOR_FREE` / `SPECTATOR_FOLLOW` / `SPECTATOR_SCOREBOARD` |
| `bot_settings_t` | struct | Bot character file path, skill level, and team assignment |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `level` | `level_locals_t` | global (extern) | Singleton for all current-map game state |
| `g_entities` | `gentity_t[MAX_GENTITIES]` | global (extern) | Flat array of all server entities; registered with engine via `trap_LocateGameData` |
| `g_gametype` … `g_proxMineTimeout` | `vmCvar_t` (many) | global (extern) | All game-configurable console variables (gravity, fraglimit, warmup, etc.) |

## Key Functions / Methods

### trap_LocateGameData
- Signature: `void trap_LocateGameData(gentity_t *gEnts, int numGEntities, int sizeofGEntity_t, playerState_t *gameClients, int sizeofGameClient)`
- Purpose: Registers the game's entity and client arrays with the engine so the server can access them directly without going through the VM call interface.
- Inputs: Pointers and sizes for `g_entities` and the client `playerState_t` array.
- Outputs/Return: void
- Side effects: Engine stores direct pointers into game memory.
- Calls: Engine syscall `G_LOCATE_GAME_DATA`.
- Notes: Must be called at `GAME_INIT`; layout of `gentity_t` and `gclient_t` first fields is ABI-contractual.

### trap_Trace
- Signature: `void trap_Trace(trace_t *results, const vec3_t start, const vec3_t mins, const vec3_t maxs, const vec3_t end, int passEntityNum, int contentmask)`
- Purpose: Performs a swept-box collision trace against all linked world geometry and entities.
- Inputs: Ray endpoints, AABB half-extents, entity to skip, content filter mask.
- Outputs/Return: Fills `trace_t` with fraction, hit plane, entity number, surface flags.
- Side effects: None (read-only query).
- Calls: Engine syscall `G_TRACE`.
- Notes: Primary spatial query used by weapons, movement, and damage code.

### trap_SendServerCommand
- Signature: `void trap_SendServerCommand(int clientNum, const char *text)`
- Purpose: Reliably sends a command string to one client (`clientNum >= 0`) or all clients (`clientNum == -1`).
- Inputs: Target client index, command string.
- Outputs/Return: void
- Side effects: Queues data in the server's reliable command channel.
- Calls: Engine syscall `G_SEND_SERVER_COMMAND`.

### trap_BotLibStartFrame / trap_AAS_* / trap_EA_* / trap_Bot*
- These form the complete bot library syscall surface (~100 functions). They delegate to the BotLib module for AAS pathfinding queries, entity action simulation, goal/move/weapon state management, and chat.
- Notes: All opaque-handle based; game code allocates states and passes integer handles back per frame.

**Notes on trivial helpers declared here:**
- `tv()` / `vtos()` — temporary vec3 formatting utilities for debug printing.
- `FOFS(x)` macro — computes byte offset of a field in `gentity_t` for `G_Find` field-search calls.

## Control Flow Notes

`g_local.h` is included by every `.c` file in the game module. It does not contain executable code. The structures it defines participate in all engine lifecycle phases:
- **Init**: `level_locals_t` is zeroed and populated; `g_entities` registered via `trap_LocateGameData`.
- **Frame** (`GAME_RUN_FRAME`): `level.time` / `level.framenum` advance; `gentity_t.think`, `.touch`, `.use`, `.pain`, `.die` callbacks are invoked.
- **Client frames** (`GAME_CLIENT_THINK`): `gclient_t.ps` is updated by pmove; `clientPersistant_t` / `clientSession_t` survive respawn/level boundaries.
- **Shutdown**: `clientSession_t` is serialized to cvars via `G_WriteSessionData`.

## External Dependencies

- **Includes**: `q_shared.h` (base types, math, `entityState_t`, `playerState_t`), `bg_public.h` (shared game types: items, weapons, pmove, events), `g_public.h` (engine API enum, `sharedEntity_t`, `entityShared_t`), `g_team.h` (CTF/team function prototypes)
- **Defined elsewhere**:
  - `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t` — `q_shared.h`
  - `entityShared_t`, `gameImport_t` — `g_public.h`
  - `gitem_t`, `weapon_t`, `team_t`, `gametype_t` — `bg_public.h`
  - All `trap_*` function bodies — `g_syscalls.c` (VM syscall dispatch stubs)
  - `level`, `g_entities`, all `vmCvar_t` definitions — `g_main.c`
