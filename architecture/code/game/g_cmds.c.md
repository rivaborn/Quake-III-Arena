# code/game/g_cmds.c

## File Purpose
Implements all client-side command handlers for the Quake III Arena game module. It serves as the primary dispatcher (`ClientCommand`) that maps incoming client command strings to their respective handler functions, covering chat, team management, voting, spectating, and cheat commands.

## Core Responsibilities
- Parse and dispatch client commands via `ClientCommand`
- Handle chat and voice communication (`say`, `say_team`, `tell`, voice variants)
- Manage team assignment and spectator follow modes
- Implement cheat commands (god, noclip, notarget, give)
- Implement voting system (callvote, vote, callteamvote, teamvote)
- Build and send scoreboard data to clients
- Handle taunt/voice chat logic including context-aware insult selection

## Key Types / Data Structures
None defined in this file. Uses types from `g_local.h`.

## Global / File-Static State
| Name | Type | Scope (global/static) | Purpose |
|---|---|---|---|
| `gc_orders` | `static char *[]` | static | Predefined game command order strings for `Cmd_GameCommand_f` |

## Key Functions / Methods

### ClientCommand
- **Signature:** `void ClientCommand( int clientNum )`
- **Purpose:** Central dispatcher; reads the command string and calls the appropriate `Cmd_*` handler.
- **Inputs:** `clientNum` — index into `g_entities` and `level.clients`
- **Outputs/Return:** void
- **Side effects:** Sends server commands to clients; may modify game state via delegated handlers
- **Calls:** `trap_Argv`, `Q_stricmp`, all `Cmd_*` functions in this file, `trap_SendServerCommand`
- **Notes:** Commands arriving during intermission are redirected to `Cmd_Say_f`. Unknown commands send an error print back to the client.

### DeathmatchScoreboardMessage
- **Signature:** `void DeathmatchScoreboardMessage( gentity_t *ent )`
- **Purpose:** Builds and sends the `scores` server command containing full scoreboard data for all connected clients.
- **Inputs:** `ent` — the requesting client entity
- **Outputs/Return:** void
- **Side effects:** Calls `trap_SendServerCommand` with a packed score string
- **Calls:** `trap_SendServerCommand`, `va`, `Com_sprintf`, `strlen`, `strcpy`
- **Notes:** Output string is capped at 1024 bytes; iteration stops early if the buffer would overflow.

### SetTeam
- **Signature:** `void SetTeam( gentity_t *ent, char *s )`
- **Purpose:** Executes a team change for a client, handling death, leader assignment, and team balance enforcement.
- **Inputs:** `ent` — client entity; `s` — team name string ("red", "blue", "spectator", "follow1", etc.)
- **Outputs/Return:** void
- **Side effects:** Modifies `client->sess`, calls `player_die`, `CopyToBodyQue`, `BroadcastTeamChange`, `ClientUserinfoChanged`, `ClientBegin`, `SetLeader`, `CheckTeamLeader`
- **Calls:** `Q_stricmp`, `PickTeam`, `TeamCount`, `player_die`, `CopyToBodyQue`, `BroadcastTeamChange`, `ClientUserinfoChanged`, `ClientBegin`, `SetLeader`, `CheckTeamLeader`, `trap_SendServerCommand`
- **Notes:** Forces spectator status in tournament mode when 2 players are already active. Enforces team balance if `g_teamForceBalance` is set.

### G_Say
- **Signature:** `void G_Say( gentity_t *ent, gentity_t *target, int mode, const char *chatText )`
- **Purpose:** Formats and broadcasts a chat message to appropriate recipients (all, team, or tell).
- **Inputs:** `ent` — sender; `target` — specific recipient or NULL for broadcast; `mode` — SAY_ALL/SAY_TEAM/SAY_TELL; `chatText` — message content
- **Outputs/Return:** void
- **Side effects:** Calls `trap_SendServerCommand` for each recipient; logs via `G_LogPrintf`; prints to console if dedicated server
- **Calls:** `G_LogPrintf`, `Team_GetLocationMsg`, `Com_sprintf`, `Q_strncpyz`, `G_SayTo`, `G_Printf`
- **Notes:** Downgrades `SAY_TEAM` to `SAY_ALL` in non-team game modes. Message text is capped at `MAX_SAY_TEXT`.

### Cmd_CallVote_f
- **Signature:** `void Cmd_CallVote_f( gentity_t *ent )`
- **Purpose:** Initiates a server-wide vote, validating the command and setting up vote state in `level`.
- **Inputs:** `ent` — calling client entity
- **Outputs/Return:** void
- **Side effects:** Modifies `level.voteTime`, `level.voteYes/No`, `level.voteString/DisplayString`; calls `trap_SetConfigstring` for vote UI
- **Calls:** `trap_Argv`, `Q_stricmp`, `strchr`, `trap_SendServerCommand`, `trap_Cvar_VariableStringBuffer`, `Com_sprintf`, `trap_SetConfigstring`, `trap_SendConsoleCommand`
- **Notes:** Semicolons in vote arguments are rejected to prevent command injection. Handles special formatting for `g_gametype` (validates range), `map` (preserves nextmap), and `nextmap` votes.

### Cmd_VoiceTaunt_f
- **Signature:** `static void Cmd_VoiceTaunt_f( gentity_t *ent )`
- **Purpose:** Selects and sends a context-sensitive voice taunt based on recent kill/death events or team praise.
- **Inputs:** `ent` — the taunting client entity
- **Outputs/Return:** void
- **Side effects:** Calls `G_Voice` to send voice commands; clears `ent->enemy` and `lastkilled_client` state
- **Calls:** `G_Voice`, `g_entities` access
- **Notes:** Priority: death insult → kill insult (gauntlet special case) → teammate praise → generic taunt.

### StopFollowing
- **Signature:** `void StopFollowing( gentity_t *ent )`
- **Purpose:** Drops a spectator out of follow mode into free spectator mode.
- **Inputs:** `ent` — spectating client entity
- **Outputs/Return:** void
- **Side effects:** Modifies `client->ps`, `client->sess`, clears `PMF_FOLLOW` and `SVF_BOT` flags

### Notes (minor helpers)
- `CheatsOk` — validates cheat prerequisites (g_cheats cvar, player alive).
- `ConcatArgs` — joins argv tokens from a start index into a static buffer.
- `SanitizeString` — strips color codes and control chars, lowercases for name comparison.
- `ClientNumberFromString` — resolves a slot number or name to a client index.
- `BroadcastTeamChange` — sends a center-print team join message to all clients.

## Control Flow Notes
`ClientCommand` is the sole engine-facing entry point, called by the server each frame when a client sends a command. It runs synchronously within the game module's command processing path. No per-frame tick logic lives here; all functions are event-driven responses to client input.

## External Dependencies
- **Includes:** `g_local.h` (all game types, trap functions, globals), `../../ui/menudef.h` (VOICECHAT_* string constants)
- **Defined elsewhere:** `level` (`level_locals_t` global from `g_main.c`), `g_entities` array, all `trap_*` syscalls (resolved by the engine VM), `player_die`, `BeginIntermission`, `TeleportPlayer`, `CopyToBodyQue`, `ClientUserinfoChanged`, `ClientBegin`, `SetLeader`, `CheckTeamLeader`, `PickTeam`, `TeamCount`, `TeamLeader`, `OnSameTeam`, `Team_GetLocationMsg`, `BG_FindItem`, `G_Spawn`, `G_SpawnItem`, `FinishSpawningItem`, `Touch_Item`, `G_FreeEntity`, `G_LogPrintf`, `G_Printf`, `G_Error`
