# code/game/g_bot.c

## File Purpose
Manages bot lifecycle within the game module: loading bot/arena definitions from data files, adding/removing bots dynamically, maintaining a deferred spawn queue, and enforcing minimum player counts per game type.

## Core Responsibilities
- Parse and cache bot info records from `scripts/bots.txt` and `.bot` files
- Parse and cache arena info records from `scripts/arenas.txt` and `.arena` files
- Allocate client slots and build userinfo strings when adding a bot (`G_AddBot`)
- Maintain a fixed-depth spawn queue (`botSpawnQueue`) to stagger bot `ClientBegin` calls
- Enforce `bot_minplayers` cvar by adding/removing random bots each 10-second interval
- Expose server commands `addbot` and `botlist` via `Svcmd_AddBot_f` / `Svcmd_BotList_f`
- Initialize single-player mode bots with correct fraglimit/timelimit from arena info

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `botSpawnQueue_t` | struct | Pairs a `clientNum` with a future `spawnTime` for deferred `ClientBegin` |
| `bot_settings_t` | struct (defined in g_local.h) | Character file path, skill, and team passed to `BotAISetupClient` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `g_numBots` | `static int` | file-static | Count of loaded bot info records |
| `g_botInfos` | `static char*[MAX_BOTS]` | file-static | Pointers to heap-allocated bot info strings |
| `g_numArenas` | `int` | global | Count of loaded arena info records |
| `g_arenaInfos` | `static char*[MAX_ARENAS]` | file-static | Pointers to heap-allocated arena info strings |
| `botSpawnQueue` | `static botSpawnQueue_t[BOT_SPAWN_QUEUE_DEPTH]` | file-static | Ring of pending deferred bot spawns |
| `bot_minplayers` | `vmCvar_t` | global | Cvar mirror for `bot_minplayers` |

## Key Functions / Methods

### G_ParseInfos
- **Signature:** `int G_ParseInfos( char *buf, int max, char *infos[] )`
- **Purpose:** Tokenizes a `{ key value … }` text block into info strings and stores them in `infos[]`
- **Inputs:** Raw text buffer, maximum entries allowed, output pointer array
- **Outputs/Return:** Number of records parsed; entries written into `infos[]` via `G_Alloc`
- **Side effects:** Heap-allocates each info string via `G_Alloc`
- **Calls:** `COM_Parse`, `COM_ParseExt`, `Info_SetValueForKey`, `G_Alloc`
- **Notes:** Allocates extra space for a `\num\` key appended later

### G_LoadBots / G_LoadArenas
- **Signature:** `static void G_LoadBots(void)` / `static void G_LoadArenas(void)`
- **Purpose:** Enumerate and load all bot/arena definition files; populate the global info arrays
- **Inputs:** None (reads cvars `g_botsFile` / `g_arenasFile` for override paths)
- **Outputs/Return:** None; updates `g_numBots`/`g_numArenas` and their info arrays
- **Side effects:** File I/O via `trap_FS_*`; registers read-only cvars
- **Calls:** `G_LoadBotsFromFile`, `G_LoadArenasFromFile`, `trap_FS_GetFileList`, `trap_Cvar_Register`
- **Notes:** `G_LoadArenas` also stamps each record with `\num\<n>` after loading

### G_AddBot
- **Signature:** `static void G_AddBot( const char *name, float skill, const char *team, int delay, char *altname )`
- **Purpose:** Fully instantiate one bot: look up its info, build userinfo, allocate a client slot, connect it, and optionally defer `ClientBegin`
- **Inputs:** Bot name (matches `g_botInfos`), skill level, team string, spawn delay ms, optional display name override
- **Outputs/Return:** void; bot is live or queued on success
- **Side effects:** Calls `trap_BotAllocateClient`, `trap_SetUserinfo`, `ClientConnect`, `ClientBegin` or `AddBotToSpawnQueue`; sets `SVF_BOT` flag on entity
- **Calls:** `G_GetBotInfoByName`, `Info_ValueForKey`, `Info_SetValueForKey`, `PickTeam`, `ClientConnect`, `ClientBegin`, `AddBotToSpawnQueue`
- **Notes:** Returns silently on any allocation failure; handicap is set based on skill bracket

### G_CheckBotSpawn
- **Signature:** `void G_CheckBotSpawn( void )`
- **Purpose:** Per-frame pump: flush any queued bots whose `spawnTime` has elapsed and enforce minimum player counts
- **Inputs:** None (reads `level.time`)
- **Outputs/Return:** void
- **Side effects:** Calls `ClientBegin` for ready entries; plays intro sound in SP mode; calls `G_CheckMinimumPlayers`
- **Calls:** `G_CheckMinimumPlayers`, `ClientBegin`, `trap_GetUserinfo`, `PlayerIntroSound`

### G_CheckMinimumPlayers
- **Signature:** `void G_CheckMinimumPlayers( void )`
- **Purpose:** Throttled (10 s) check that adds or removes random bots to satisfy `bot_minplayers`; branches on game type
- **Inputs:** None
- **Side effects:** May call `G_AddRandomBot` or `G_RemoveRandomBot`; updates static `checkminimumplayers_time`

### G_BotConnect
- **Signature:** `qboolean G_BotConnect( int clientNum, qboolean restart )`
- **Purpose:** Called from `ClientConnect` path for bots; extracts settings from userinfo and calls `BotAISetupClient`
- **Inputs:** Client slot number, restart flag
- **Outputs/Return:** `qtrue` on success; `qfalse` drops the client
- **Side effects:** Calls `BotAISetupClient`; may call `trap_DropClient`

### G_InitBots
- **Signature:** `void G_InitBots( qboolean restart )`
- **Purpose:** Entry point called at map start; loads data files, registers `bot_minplayers`, and in SP mode reads arena info to set limits and spawn the bot roster
- **Inputs:** `restart` — suppresses `G_SpawnBots` on map restart
- **Side effects:** Sets `fraglimit`/`timelimit` cvars in SP mode; calls `G_SpawnBots` via console command injection

## Control Flow Notes
- **Init:** `G_InitBots` is called once during `G_InitGame` (g_main.c).
- **Frame:** `G_CheckBotSpawn` is called every server frame from `G_RunFrame` to drain the spawn queue and recheck player counts.
- **Connect:** `G_BotConnect` is called by `ClientConnect` when the connecting client has `SVF_BOT` set.
- **Disconnect:** `G_RemoveQueuedBotBegin` is called from `ClientDisconnect` to cancel any pending spawn.

## External Dependencies
- `g_local.h` — all shared game types, trap declarations, `level`, `g_entities`, cvars
- `BotAISetupClient`, `BotAIShutdown` — defined in `ai_main.c` (botlib AI layer)
- `ClientConnect`, `ClientBegin`, `ClientDisconnect` — defined in `g_client.c`
- `G_Alloc` — defined in `g_mem.c`
- `PickTeam` — defined in `g_client.c`
- `podium1/2/3` — `extern gentity_t*` owned by `g_arenas.c`
- `COM_Parse`, `COM_ParseExt`, `Info_SetValueForKey`, `Info_ValueForKey`, `Q_strncpyz` — defined in `q_shared.c` / `bg_lib.c`
- All `trap_*` functions — syscall stubs resolved by the VM/engine boundary
