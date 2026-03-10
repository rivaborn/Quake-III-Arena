# code/game/ai_chat.c

## File Purpose
Implements the bot chat/taunting AI layer for Quake III Arena. It decides when and what a bot says in response to game events (entering/exiting a game, kills, deaths, random chatter, etc.), gating all output behind cooldown timers, game-mode checks, and bot personality characteristics.

## Core Responsibilities
- Enforce chat rate-limiting via `TIME_BETWEENCHATTING` (25 s cooldown)
- Query player rankings and opponent lists to populate chat template variables
- Validate whether a bot is in a safe position to chat (not in lava/water, on solid ground, no active powerups, no visible enemies)
- Select the appropriate chat category string (e.g., `"death_rail"`, `"kill_insult"`) based on game context and random characteristic weights
- Delegate actual message construction and queuing to `BotAI_BotInitialChat` / `trap_BotEnterChat`
- Issue `vtaunt` voice commands in team-play modes instead of text chat
- Provide `BotChatTest` to exhaustively exercise all chat categories for debugging

## Key Types / Data Structures
None defined in this file. Uses `bot_state_t` (defined in `ai_main.h`), `aas_entityinfo_t`, `playerState_t`, and `bsp_trace_t` from included headers.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `TIME_BETWEENCHATTING` | `#define` (25) | file | Minimum seconds between any two bot chat events |
| `maxclients` (in multiple functions) | `static int` | per-function static | Cached value of `sv_maxclients` cvar; lazily initialized on first call |

## Key Functions / Methods

### BotNumActivePlayers
- **Signature:** `int BotNumActivePlayers(void)`
- **Purpose:** Count non-spectator, named players currently in the game.
- **Inputs:** None (reads config strings via `trap_GetConfigstring`)
- **Outputs/Return:** Player count as `int`
- **Side effects:** Lazily caches `sv_maxclients` in a function-local static.
- **Calls:** `trap_GetConfigstring`, `trap_Cvar_VariableIntegerValue`, `Info_ValueForKey`, `strlen`, `atoi`
- **Notes:** Used by every `BotChat_*` function to suppress chat when fewer than 2 active players exist.

### BotValidChatPosition
- **Signature:** `int BotValidChatPosition(bot_state_t *bs)`
- **Purpose:** Determine whether the bot is in a safe, idle position where chatting is appropriate.
- **Inputs:** `bs` — current bot state
- **Outputs/Return:** `qtrue` if valid, `qfalse` otherwise
- **Side effects:** Issues a BSP trace via `BotAI_Trace`.
- **Calls:** `BotIsDead`, `trap_PointContents`, `trap_AAS_PresenceTypeBoundingBox`, `BotAI_Trace`, `VectorCopy`
- **Notes:** Rejects chatting with any active powerup (Quad, Haste, Invis, Regen, Flight), in lava/slime/water, or when not standing on the world entity.

### BotChat_Death
- **Signature:** `int BotChat_Death(bot_state_t *bs)`
- **Purpose:** Select and queue an appropriate death-reaction chat message based on cause of death (`botdeathtype`) and killer identity.
- **Inputs:** `bs` — bot state with `botdeathtype`, `lastkilledby`, `botsuicide` populated
- **Outputs/Return:** `qtrue` if a message was queued
- **Side effects:** Sets `bs->lastchat_time`, `bs->chatto`; may issue `vtaunt` command in team play.
- **Calls:** `BotAI_BotInitialChat`, `trap_Characteristic_BFloat`, `BotSameTeam`, `TeamPlayIsOn`, `trap_EA_Command`, `BotRandomOpponentName`, `BotWeaponNameForMeansOfDeath`, `trap_BotNumInitialChats`
- **Notes:** Most complex branching in the file; handles ~12 death sub-categories including kamikaze (MissionPack only).

### BotChat_Kill
- **Signature:** `int BotChat_Kill(bot_state_t *bs)`
- **Purpose:** Queue a kill-reaction chat after the bot kills another player.
- **Inputs:** `bs` — bot state with `lastkilledplayer`, `enemydeathtype` set
- **Outputs/Return:** `qtrue` if message queued
- **Side effects:** Sets `bs->lastchat_time`, `bs->chatto`; may issue `vtaunt` in team play.
- **Calls:** `BotVisibleEnemies`, `BotValidChatPosition`, `BotSameTeam`, `BotAI_BotInitialChat`, `trap_EA_Command`
- **Notes:** Suppressed if enemies are still visible (bot not safe to taunt).

### BotChat_Random
- **Signature:** `int BotChat_Random(bot_state_t *bs)`
- **Purpose:** Emit unsolicited random chatter proportional to think time.
- **Inputs:** `bs`
- **Outputs/Return:** `qtrue` if message queued
- **Side effects:** Sets `bs->lastchat_time`, `bs->chatto`
- **Calls:** `BotAI_BotInitialChat`, `trap_Characteristic_BFloat`, `BotRandomOpponentName`, `BotMapTitle`, `BotRandomWeaponName`
- **Notes:** Skipped during LTG_TEAMHELP / LTG_TEAMACCOMPANY / LTG_RUSHBASE objectives.

### BotChatTest
- **Signature:** `void BotChatTest(bot_state_t *bs)`
- **Purpose:** Debug utility that iterates and sends every chat category.
- **Inputs:** `bs`
- **Outputs/Return:** void
- **Side effects:** Calls `trap_BotEnterChat` for each category, directly transmitting to all clients.
- **Notes:** Covers all ~20 chat types; not guarded by cooldown or position checks.

### Notes (minor helpers)
- `BotIsFirstInRankings` / `BotIsLastInRankings` — compare bot score against all active players.
- `BotFirstClientInRankings` / `BotLastClientInRankings` — return static name buffers of highest/lowest-scoring players.
- `BotRandomOpponentName` — picks a random non-teammate opponent name into a static buffer.
- `BotMapTitle` — extracts `mapname` from server info into a static buffer.
- `BotWeaponNameForMeansOfDeath` — maps `MOD_*` enum to a display string.
- `BotRandomWeaponName` — returns a random weapon name string for flavor text.
- `BotChatTime` — returns a constant `2.0` (CPM calculation is commented out).
- `BotVisibleEnemies` — scans all clients for visible, living, non-teammate enemies.

## Control Flow Notes
This file is called reactively (event-driven), not on every frame. Each `BotChat_*` function is invoked from higher-level AI state management (e.g., `ai_main.c`, `ai_dmq3.c`) when the corresponding game event occurs (spawn, death, kill, level end). `BotChat_Random` is the only one with a frame-time-proportional trigger (`bs->thinktime * 0.1`). All functions return immediately (`qfalse`) if the global `bot_nochat` cvar is set.

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_char.h`, `be_ai_chat.h`, `be_ai_gen.h`, `be_ai_goal.h`, `be_ai_move.h`, `be_ai_weap.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`; conditionally `ui/menudef.h` (MissionPack)
- **Defined elsewhere:** `bot_state_t`, `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotAI_Trace`, `EasyClientName`, `ClientName`, `BotEntityInfo`, `BotEntityVisible`, `BotSameTeam`, `BotIsDead`, `BotIsObserver`, `EntityIsDead`, `EntityIsInvisible`, `EntityIsShooting`, `TeamPlayIsOn`, `FloatTime`, `gametype`, `bot_nochat`, `bot_fastchat`, `g_entities`, all `trap_*` syscalls
