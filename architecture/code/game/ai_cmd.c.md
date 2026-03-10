# code/game/ai_cmd.c

## File Purpose
Implements the bot AI command-processing layer for Quake III Arena's team-play modes. It parses structured natural-language chat matches (e.g. "help me", "defend the flag") received from human players and translates them into long-term goal (LTG) state changes on the receiving `bot_state_t`. It is the bridge between the bot chat-matching subsystem and the bot goal/behavior system.

## Core Responsibilities
- Receive a raw chat string via `BotMatchMessage`, classify it against known message templates (`trap_BotFindMatch`), and dispatch to a typed handler.
- Determine whether a match message is actually addressed to this bot (`BotAddressedToBot`).
- Resolve named teammates, enemies, map items, and waypoints from human-readable strings into engine-usable identifiers.
- Set `bs->ltgtype`, `bs->teamgoal`, `bs->teamgoal_time`, and related fields on the bot state to steer high-level behavior.
- Manage bot sub-team membership, team-leader tracking, and the `notleader[]` flag array.
- Parse and store patrol waypoint chains and user-defined checkpoint waypoints.
- Track CTF/1FCTF flag status changes reported through team chat.

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bot_state_t` | struct (defined in `ai_main.h`) | Per-bot AI state; central data modified by every handler in this file |
| `bot_match_t` | struct (defined in botlib) | Result of `trap_BotFindMatch`; carries message type, subtype, and named variable slots |
| `bot_goal_t` | struct (defined in botlib) | AAS-linked goal (entity, area, origin, bounds) assigned as a team goal |
| `bot_waypoint_t` | struct (defined in `ai_main.h`) | Named checkpoint/patrol node with a linked `bot_goal_t` |
| `aas_entityinfo_t` | struct (`be_aas.h`) | Spatial snapshot of an entity used to bootstrap a team goal from a live client |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `notleader` | `int[MAX_CLIENTS]` | global | Per-client flag suppressing re-election as team leader after explicit resignation |

## Key Functions / Methods

### BotMatchMessage
- **Signature:** `int BotMatchMessage(bot_state_t *bs, char *message)`
- **Purpose:** Top-level entry point. Classifies a chat string and dispatches to the appropriate `BotMatch_*` handler.
- **Inputs:** Bot state pointer; raw chat string.
- **Outputs/Return:** `qtrue` if the message matched a known type; `qfalse` otherwise.
- **Side effects:** Mutates `bs` indirectly through called handlers.
- **Calls:** `trap_BotFindMatch`, all `BotMatch_*` functions, `BotAI_Print`.
- **Notes:** Uses `MTCONTEXT_MISC | MTCONTEXT_INITIALTEAMCHAT | MTCONTEXT_CTF` as the match context.

### BotAddressedToBot
- **Signature:** `int BotAddressedToBot(bot_state_t *bs, bot_match_t *match)`
- **Purpose:** Determines whether the matched message is directed at this specific bot (by name, sub-team, or broadcast) or should be handled by random chance when no direct addressee.
- **Inputs:** Bot state; match result.
- **Outputs/Return:** Non-zero if the bot should respond.
- **Side effects:** None.
- **Calls:** `trap_BotMatchVariable`, `ClientOnSameTeamFromName`, `ClientName`, `trap_BotFindMatch`, `stristr`, `NumPlayersOnSameTeam`, `random`.

### BotGetPatrolWaypoints
- **Signature:** `int BotGetPatrolWaypoints(bot_state_t *bs, bot_match_t *match)`
- **Purpose:** Parses a multi-segment patrol order string into a linked list of `bot_waypoint_t` nodes and stores them in `bs->patrolpoints`.
- **Inputs:** Bot state; match containing `KEYAREA` variable chain.
- **Outputs/Return:** `qtrue` on success.
- **Side effects:** Allocates waypoints via `BotCreateWayPoint`; frees old patrol points via `BotFreeWaypoints`; modifies `bs->patrolpoints`, `bs->curpatrolpoint`, `bs->patrolflags`.
- **Calls:** `trap_BotMatchVariable`, `trap_BotFindMatch`, `BotGetMessageTeamGoal`, `BotCreateWayPoint`, `BotFreeWaypoints`, `trap_EA_SayTeam`.

### BotMatch_HelpAccompany
- **Signature:** `void BotMatch_HelpAccompany(bot_state_t *bs, bot_match_t *match)`
- **Purpose:** Handles `MSG_HELP` and `MSG_ACCOMPANY`; sets `bs->ltgtype` to `LTG_TEAMHELP` or `LTG_TEAMACCOMPANY` and resolves the teammate's current AAS location.
- **Side effects:** Modifies `bs->ltgtype`, `bs->teamgoal`, `bs->teammate`, `bs->decisionmaker`, `bs->ordered`, `bs->teamgoal_time`, `bs->formation_dist`; may send chat.
- **Calls:** `BotAddressedToBot`, `BotEntityInfo`, `BotPointAreaNum`, `BotGetTime`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`, `trap_BotEnterChat`, `BotAI_BotInitialChat`.

### BotMatch_Camp
- **Signature:** `void BotMatch_Camp(bot_state_t *bs, bot_match_t *match)`
- **Purpose:** Handles `MSG_CAMP`; resolves camp location (ST_THERE = bot's current pos, ST_HERE = requester's pos, or named item) and sets `LTG_CAMPORDER`.
- **Side effects:** Modifies `bs->ltgtype`, `bs->teamgoal`, `bs->teamgoal_time`, `bs->arrive_time`, `bs->ordered`, `bs->decisionmaker`.
- **Calls:** `BotEntityInfo`, `BotPointAreaNum`, `BotGetMessageTeamGoal`, `BotGetTime`, `BotSetTeamStatus`.

### BotMatch_CTF
- **Signature:** `void BotMatch_CTF(bot_state_t *bs, bot_match_t *match)`
- **Purpose:** Tracks flag-picked-up, flag-captured, and flag-returned events to update `bs->redflagstatus`, `bs->blueflagstatus`, `bs->flagcarrier`, and `bs->flagstatuschanged`.
- **Side effects:** Writes flag status fields on `bs`; sets `bs->lastflagcapture_time`.
- **Calls:** `trap_BotMatchVariable`, `ClientFromName`.

### BotGetItemTeamGoal / BotGetMessageTeamGoal
- **Notes:** `BotGetItemTeamGoal` iterates level items by name via `trap_BotGetLevelItemGoal`, skipping dropped items. `BotGetMessageTeamGoal` extends that by also checking the bot's known checkpoint list (`BotFindWayPoint`). Both return a populated `bot_goal_t`.

### BotNearestVisibleItem
- **Signature:** `float BotNearestVisibleItem(bot_state_t *bs, char *itemname, bot_goal_t *goal)`
- **Purpose:** Finds the closest level item by name that has an unobstructed LOS from the bot's eye position.
- **Outputs/Return:** Distance to the nearest visible item; fills `goal`.
- **Calls:** `trap_BotGetLevelItemGoal`, `trap_BotGoalName`, `BotAI_Trace`.

## Control Flow Notes
This file is driven entirely by the bot frame loop in `ai_main.c`/`ai_dmnet.c`. Each frame, the bot reads its console message queue; any message not handled by the Eliza chat system is passed to `BotMatchMessage`. There is no init or shutdown path in this file. The handlers only write to `bot_state_t`; the updated LTG type is acted on in the next frame by the bot's goal/movement subsystem.

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_char/chat/gen/goal/move/weap.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`, `ui/menudef.h`
- **Defined elsewhere (used here):** `bot_state_t`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`, `BotGetAlternateRouteGoal`, `BotOppositeTeam`, `BotSameTeam`, `BotTeam`, `BotFindWayPoint`, `BotCreateWayPoint`, `BotFreeWaypoints`, `BotVoiceChat`, `BotVoiceChatOnly`, `TeamPlayIsOn`, `ClientFromName`, `ClientOnSameTeamFromName`, `EasyClientName`, `BotAI_BotInitialChat`, `BotAI_Trace`, `FloatTime`, `gametype`, `ctf_redflag`, `ctf_blueflag`, all `trap_*` syscalls.
