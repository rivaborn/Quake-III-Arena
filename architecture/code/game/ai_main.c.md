# code/game/ai_main.c

## File Purpose
Central bot AI management module for Quake III Arena. It handles bot lifecycle (setup, shutdown, per-frame thinking), bridges the game module and the botlib, and converts bot AI decisions into usercmd_t inputs submitted to the server.

## Core Responsibilities
- Initialize and shut down the bot library and per-bot state (`BotAISetup`, `BotAIShutdown`, `BotAISetupClient`, `BotAIShutdownClient`)
- Drive per-frame bot thinking via `BotAIStartFrame`, dispatching `BotAI` for each active bot
- Translate bot inputs (`bot_input_t`) into network-compatible `usercmd_t` commands
- Manage view-angle interpolation and smoothing for bots
- Feed entity state updates into the botlib each frame
- Implement bot interbreeding (genetic algorithm) for fuzzy-logic goal evolution
- Persist and restore per-bot session data across map restarts

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_state_t` | struct (defined in `ai_main.h`) | Per-bot runtime state: player state, goals, move/goal/chat/weapon handles, view angles, etc. |
| `bot_settings_t` | struct | Configuration for a single bot: character file, skill, team |
| `bot_input_t` | struct (botlib.h) | Botlib output: movement dir, speed, view angles, action flags, weapon |
| `bot_entitystate_t` | struct (botlib.h) | Entity state snapshot fed into the botlib each frame |
| `bsp_trace_t` | struct (botlib.h) | Trace result in botlib format, populated from engine `trace_t` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botstates` | `bot_state_t *[MAX_CLIENTS]` | global | Array of pointers to per-bot state, indexed by client number |
| `numbots` | `int` | global | Count of active bots |
| `floattime` | `float` | global | Current AAS floating-point time, updated each frame |
| `regularupdate_time` | `float` | global | Throttle for `BotAIRegularUpdate` (every 0.3 s) |
| `bot_interbreed` | `int` | global | Flag: interbreeding is active |
| `bot_interbreedmatchcount` | `int` | global | Match counter for interbreeding cycle |
| `bot_thinktime` | `vmCvar_t` | global | Cvar: ms between bot AI ticks (default 100) |
| `bot_pause` | `vmCvar_t` | global | Cvar: freeze bot AI |
| `bot_report` | `vmCvar_t` | global | Cvar: update bot info config strings |
| `bot_memorydump` | `vmCvar_t` | global | Cvar: trigger botlib memory dump |
| `bot_saveroutingcache` | `vmCvar_t` | global | Cvar: save AAS routing cache |
| `bot_testsolid` / `bot_testclusters` | `vmCvar_t` | global | Cvars: AAS debug visualisation |
| `bot_developer` | `vmCvar_t` | global | Cvar: bot developer mode |
| `bot_interbreedchar/bots/cycle/write` | `vmCvar_t` | global | Cvars controlling interbreeding |

## Key Functions / Methods

### BotAIStartFrame
- **Signature:** `int BotAIStartFrame(int time)`
- **Purpose:** Main per-frame entry point; updates botlib, feeds entity states, schedules and runs bot AI, submits usercmds.
- **Inputs:** `time` — current server time in ms
- **Outputs/Return:** `qtrue` on success, `qfalse` if AAS not initialized
- **Side effects:** Calls `trap_BotLibStartFrame`, `trap_BotLibUpdateEntity` for all entities, calls `BotAI` for each bot whose residual exceeds the think interval, calls `BotUpdateInput`/`trap_BotUserCommand` for every bot every frame
- **Calls:** `G_CheckBotSpawn`, `BotInterbreeding`, `BotScheduleBotThink`, `BotAIRegularUpdate`, `BotAI`, `BotUpdateInput`, `trap_BotUserCommand`, `trap_BotLibUpdateEntity`, `trap_AAS_Time`
- **Notes:** Uses a static `botlib_residual` to throttle botlib updates independently of bot think time; bot think is staggered via `botthink_residual` per bot.

### BotAI
- **Signature:** `int BotAI(int client, float thinktime)`
- **Purpose:** Per-bot AI tick: retrieves player state, processes server commands, runs deathmatch AI, selects weapon.
- **Inputs:** `client` index, `thinktime` in seconds
- **Outputs/Return:** `qtrue`/`qfalse`
- **Side effects:** Updates `bs->cur_ps`, `bs->ltime`, `bs->origin`, `bs->eye`, `bs->areanum`; queues console messages; calls `BotDeathmatchAI`; calls `trap_EA_SelectWeapon`
- **Calls:** `BotAI_GetClientState`, `trap_BotGetServerCommand`, `BotVoiceChatCommand` (MISSIONPACK), `BotDeathmatchAI`, `BotPointAreaNum`, `trap_EA_SelectWeapon`

### BotAISetupClient
- **Signature:** `int BotAISetupClient(int client, struct bot_settings_s *settings, qboolean restart)`
- **Purpose:** Allocates and initializes all botlib subsystem handles (goal, weapon, chat, move states) for one bot.
- **Inputs:** `client` index, `settings`, `restart` flag
- **Outputs/Return:** `qtrue` on success
- **Side effects:** Allocates `bot_state_t` via `G_Alloc`; increments `numbots`; calls `BotScheduleBotThink`; may call `BotReadSessionData` on restart
- **Calls:** `trap_BotLoadCharacter`, `trap_BotAllocGoalState`, `trap_BotLoadItemWeights`, `trap_BotAllocWeaponState`, `trap_BotAllocChatState`, `trap_BotAllocMoveState`, `BotChatTest`, `BotScheduleBotThink`

### BotAIShutdownClient
- **Signature:** `int BotAIShutdownClient(int client, qboolean restart)`
- **Purpose:** Frees all botlib handles and clears `bot_state_t` for one bot.
- **Side effects:** Decrements `numbots`; may call `BotWriteSessionData`; frees move/goal/chat/weapon states and character
- **Calls:** `BotWriteSessionData`, `BotChat_ExitGame`, `trap_BotEnterChat`, `trap_BotFree*`, `BotFreeWaypoints`, `BotClearActivateGoalStack`

### BotAISetup
- **Signature:** `int BotAISetup(int restart)`
- **Purpose:** Registers all bot-related cvars and calls `BotInitLibrary` to set up the botlib (skipped on tournament restart).
- **Calls:** `trap_Cvar_Register`, `BotInitLibrary`

### BotAIShutdown
- **Signature:** `int BotAIShutdown(int restart)`
- **Purpose:** Shuts down all bot clients; on full shutdown also shuts down the botlib.
- **Calls:** `BotAIShutdownClient`, `trap_BotLibShutdown`

### BotAILoadMap
- **Signature:** `int BotAILoadMap(int restart)`
- **Purpose:** Loads the AAS map into botlib and resets all active bot states.
- **Calls:** `trap_BotLibLoadMap`, `BotResetState`, `BotSetupDeathmatchAI`

### BotInputToUserCommand
- **Signature:** `void BotInputToUserCommand(bot_input_t *bi, usercmd_t *ucmd, int delta_angles[3], int time)`
- **Purpose:** Converts botlib `bot_input_t` (direction vector, speed, action flags) into a `usercmd_t` suitable for submission to the server.
- **Side effects:** Writes into `*ucmd`
- **Notes:** Speed is scaled from [0,400] to [0,127]; movement is projected onto view-relative forward/right axes.

### BotUpdateInput
- **Signature:** `void BotUpdateInput(bot_state_t *bs, int time, int elapsed_time)`
- **Purpose:** Applies delta angles, calls `BotChangeViewAngles`, retrieves EA input, converts to usercmd.
- **Calls:** `BotChangeViewAngles`, `trap_EA_GetInput`, `BotInputToUserCommand`

### BotChangeViewAngles
- **Signature:** `void BotChangeViewAngles(bot_state_t *bs, float thinktime)`
- **Purpose:** Smoothly interpolates bot view angles toward `ideal_viewangles` using either smooth or over-reaction model depending on `bot_challenge`.
- **Side effects:** Modifies `bs->viewangles`, calls `trap_EA_View`

### BotInterbreedBots / BotInterbreeding
- **Purpose:** Genetic algorithm support — `BotInterbreeding` activates tournament-mode interbreeding by respawning bots; `BotInterbreedBots` uses kill/death ratios as fitness ranks to cross-breed fuzzy logic goal weights.
- **Calls:** `trap_GeneticParentsAndChildSelection`, `trap_BotInterbreedGoalFuzzyLogic`, `trap_BotMutateGoalFuzzyLogic`

### BotAI_Trace
- **Signature:** `void BotAI_Trace(bsp_trace_t *bsptrace, ...)`
- **Purpose:** Adapter that calls `trap_Trace` and copies the result into the botlib `bsp_trace_t` format.

## Control Flow Notes

- **Init:** `BotAISetup` → `BotInitLibrary` → `trap_BotLibSetup` at game startup; `BotAILoadMap` on each map load.
- **Per-frame:** `BotAIStartFrame` is called once per server frame by `g_bot.c`/`g_main.c`. It drives the botlib update, staggered bot think ticks, view-angle update, and final usercmd submission.
- **Shutdown:** `BotAIShutdown` at game shutdown or map change.

## External Dependencies

- `g_local.h` / `g_public.h` — game entity types, trap functions, game globals (`g_entities`, `level`, `maxclients`, `gametype`)
- `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h` — botlib API: AAS, elementary actions, chat/goal/move/weapon AI
- `ai_dmq3.h` / `ai_dmnet.h` / `ai_chat.h` / `ai_cmd.h` / `ai_vcmd.h` — higher-level deathmatch AI (`BotDeathmatchAI`, `BotSetupDeathmatchAI`, `BotChat_ExitGame`, etc.)
- `chars.h`, `inv.h`, `syn.h` — bot character, inventory, and synonym constants
- `trap_*` functions — VM syscall interface to the engine (AAS, EA, BotLib, Cvar, Trace, etc.), defined elsewhere in the engine/game syscall layer
- `ExitLevel` — declared extern, defined in `g_main.c`
