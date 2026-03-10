# code/cgame/cg_servercmds.c

## File Purpose
Handles reliably-sequenced text commands sent by the server to the cgame module. All commands are processed at snapshot transition time, guaranteeing a valid snapshot is present. Also manages the voice chat system including parsing, buffering, and playback.

## Core Responsibilities
- Dispatch incoming server commands (`cp`, `cs`, `print`, `chat`, `tchat`, `scores`, `tinfo`, `map_restart`, etc.) via `CG_ServerCommand`
- Parse and apply score data (`CG_ParseScores`) and team overlay info (`CG_ParseTeamInfo`)
- Parse and cache server configuration strings (`CG_ParseServerinfo`, `CG_SetConfigValues`)
- Handle config-string change notifications and re-register models/sounds/client info accordingly
- Load, parse, and look up character voice chat files (`.voice`, `.vc`)
- Buffer and throttle voice chat playback with a ring buffer
- Handle map restarts, warmup transitions, and shader remapping

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `orderTask_t` | struct | Maps a voice chat order string to a team task number |
| `voiceChat_t` | struct | One named voice chat entry with multiple sound variants and chat strings |
| `voiceChatList_t` | struct | A full voice file (name, gender, array of `voiceChat_t`) |
| `headModelVoiceChat_t` | struct | Cache mapping a head model name to a `voiceChatLists` index |
| `bufferedVoiceChat_t` | struct | A single queued voice chat event (client, sound, message, cmd) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `validOrders` | `static const orderTask_t[]` | file-static | Table of voice order strings → team task numbers |
| `numValidOrders` | `static const int` | file-static | Count of entries in `validOrders` |
| `voiceChatLists` | `voiceChatList_t[MAX_VOICEFILES]` | global (file-level) | Loaded voice chat data for up to 8 voice files |
| `headModelVoiceChat` | `headModelVoiceChat_t[MAX_HEADMODELS]` | global (file-level) | Cache of head model → voice list index mappings |
| `voiceChatBuffer` | `bufferedVoiceChat_t[MAX_VOICECHATBUFFER]` | global (file-level) | Ring buffer for queued voice chat events |

## Key Functions / Methods

### CG_ExecuteNewServerCommands
- Signature: `void CG_ExecuteNewServerCommands( int latestSequence )`
- Purpose: Primary entry point — drains the server command queue up to `latestSequence`, calling `CG_ServerCommand` for each.
- Inputs: `latestSequence` — the most recent server command sequence number from the snapshot.
- Outputs/Return: None.
- Side effects: Advances `cgs.serverCommandSequence`; indirectly modifies all cgame state via dispatched sub-handlers.
- Calls: `trap_GetServerCommand`, `CG_ServerCommand`
- Notes: Called once per snapshot transition by the cgame frame loop.

### CG_ServerCommand
- Signature: `static void CG_ServerCommand( void )`
- Purpose: Tokenizes and dispatches the current server command string to the appropriate handler.
- Inputs: None (reads from `CG_Argv`).
- Outputs/Return: None.
- Side effects: Calls one of many sub-handlers depending on command token; falls through to an "unknown command" print.
- Calls: `CG_CenterPrint`, `CG_ConfigStringModified`, `CG_Printf`, `CG_AddToTeamChat`, `CG_VoiceChat`, `CG_ParseScores`, `CG_ParseTeamInfo`, `CG_MapRestart`, `CG_LoadDeferredPlayers`, `trap_R_RemapShader`
- Notes: Handles: `cp`, `cs`, `print`, `chat`, `tchat`, `vchat`, `vtchat`, `vtell`, `scores`, `tinfo`, `map_restart`, `remapShader`, `loaddefered`, `clientLevelShot`.

### CG_ParseServerinfo
- Signature: `void CG_ParseServerinfo( void )`
- Purpose: Reads and caches all game-rule cvars from `CS_SERVERINFO` config string into `cgs`.
- Inputs: None (reads `CS_SERVERINFO` via `CG_ConfigString`).
- Outputs/Return: None.
- Side effects: Writes `cgs.gametype`, `cgs.dmflags`, `cgs.fraglimit`, `cgs.mapname`, `cgs.redTeam`, `cgs.blueTeam`, etc.; calls `trap_Cvar_Set` to mirror values.
- Calls: `CG_ConfigString`, `Info_ValueForKey`, `trap_Cvar_Set`, `Com_sprintf`, `Q_strncpyz`

### CG_ConfigStringModified
- Signature: `static void CG_ConfigStringModified( void )`
- Purpose: Handles a `cs` server command; refreshes the full gamestate then reacts to the specific config string index that changed.
- Inputs: None (reads `CG_Argv(1)` for the CS index).
- Outputs/Return: None.
- Side effects: Updates `cgs` fields, registers new models/sounds, updates client infos, triggers sounds.
- Calls: `trap_GetGameState`, `CG_ConfigString`, `CG_StartMusic`, `CG_ParseServerinfo`, `CG_ParseWarmup`, `trap_R_RegisterModel`, `trap_S_RegisterSound`, `CG_NewClientInfo`, `CG_BuildSpectatorString`, `CG_ShaderStateChanged`, `trap_S_StartLocalSound`

### CG_MapRestart
- Signature: `static void CG_MapRestart( void )`
- Purpose: Resets transient cgame state in response to a `map_restart` server command.
- Inputs: None.
- Outputs/Return: None.
- Side effects: Clears local entities, marks, particles, resets counters and flags, starts music, clears looping sounds, plays "fight" sound if warmup is done.
- Calls: `CG_InitLocalEntities`, `CG_InitMarkPolys`, `CG_ClearParticles`, `CG_StartMusic`, `trap_S_ClearLoopingSounds`, `trap_S_StartLocalSound`, `CG_CenterPrint`, `trap_Cvar_Set`, `trap_SendConsoleCommand`

### CG_ParseVoiceChats
- Signature: `int CG_ParseVoiceChats( const char *filename, voiceChatList_t *voiceChatList, int maxVoiceChats )`
- Purpose: Reads and parses a `.voice` script file into a `voiceChatList_t`, registering all sounds.
- Inputs: `filename` — path to voice file; `voiceChatList` — output structure; `maxVoiceChats` — capacity limit.
- Outputs/Return: `qtrue` on success/EOF, `qfalse` on hard error.
- Side effects: File I/O; calls `trap_S_RegisterSound` for each sound entry; populates `voiceChatList`.
- Calls: `trap_FS_FOpenFile`, `trap_FS_Read`, `trap_FS_FCloseFile`, `COM_ParseExt`, `trap_S_RegisterSound`, `trap_Print`

### CG_VoiceChatListForClient
- Signature: `voiceChatList_t *CG_VoiceChatListForClient( int clientNum )`
- Purpose: Resolves which `voiceChatList_t` to use for a given client, using head-model → `.vc` file lookup with gender fallback.
- Inputs: `clientNum` — client index.
- Outputs/Return: Pointer into `voiceChatLists[]`.
- Side effects: May call `CG_HeadModelVoiceChats` (file I/O); writes into `headModelVoiceChat` cache.
- Calls: `CG_HeadModelVoiceChats`, `Q_stricmp`, `Com_sprintf`

### CG_PlayVoiceChat / CG_AddBufferedVoiceChat / CG_PlayBufferedVoiceChats
- These three form the voice chat ring-buffer system (MISSIONPACK only): `CG_AddBufferedVoiceChat` enqueues into `voiceChatBuffer`; `CG_PlayBufferedVoiceChats` is polled each frame and dequeues one entry per second; `CG_PlayVoiceChat` plays the sound, optionally prints to team chat, and handles order acceptance state.

### CG_AddToTeamChat
- Signature: `static void CG_AddToTeamChat( const char *str )`
- Purpose: Word-wraps a string into the team chat ring buffer in `cgs.teamChatMsgs[]`, preserving color codes across line breaks.
- Side effects: Writes `cgs.teamChatMsgs`, `cgs.teamChatMsgTimes`, advances `cgs.teamChatPos`.

## Control Flow Notes
- `CG_ExecuteNewServerCommands` is called by `cg_snapshot.c`/`CG_ProcessSnapshots` at snapshot transition time.
- `CG_ParseServerinfo` and `CG_SetConfigValues` are called explicitly during initial gamestate load from `cg_main.c`.
- `CG_PlayBufferedVoiceChats` is called every render frame from `cg_draw.c` or equivalent.
- This file has no per-frame rendering or physics involvement; it is purely event/command driven.

## External Dependencies
- `cg_local.h` — all cgame types (`cg_t`, `cgs_t`, `clientInfo_t`, trap functions, cvars)
- `ui/menudef.h` — `VOICECHAT_*` string constants and UI owner-draw defines
- **Defined elsewhere:** `CG_ConfigString`, `CG_Argv`, `CG_StartMusic`, `CG_NewClientInfo`, `CG_BuildSpectatorString`, `CG_InitLocalEntities`, `CG_InitMarkPolys`, `CG_ClearParticles`, `CG_SetScoreSelection`, `CG_ShowResponseHead`, `CG_LoadDeferredPlayers`, `COM_ParseExt`, `Info_ValueForKey`, all `trap_*` syscalls
