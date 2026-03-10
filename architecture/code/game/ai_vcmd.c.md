# code/game/ai_vcmd.c

## File Purpose
Handles bot AI responses to voice chat commands issued by human teammates. It maps incoming voice chat strings to specific bot behavioral state changes, enabling human players to direct bot teammates using in-game voice commands.

## Core Responsibilities
- Parse and dispatch incoming voice chat commands to handler functions
- Assign new long-term goal (LTG) types to bots in response to orders (get flag, defend, camp, follow, etc.)
- Validate gametype and team membership before acting on commands
- Manage bot leadership state (`teamleader`, `notleader`)
- Record task preferences for teammates (attacker vs. defender)
- Reset bot goal state when ordered to patrol (dismiss)
- Send acknowledgment chat/voice responses back to the commanding client

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `voiceCommand_t` | struct | Associates a voice chat string constant with a handler function pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `voiceCommands[]` | `voiceCommand_t[]` | file-static (file-scope array) | Dispatch table mapping voice chat string identifiers to handler functions; terminated by a `NULL` cmd sentinel |

## Key Functions / Methods

### BotVoiceChat_GetFlag
- **Signature:** `void BotVoiceChat_GetFlag(bot_state_t *bs, int client, int mode)`
- **Purpose:** Orders bot to capture the enemy flag (CTF/1FCTF only).
- **Inputs:** Bot state, commanding client index, chat mode.
- **Outputs/Return:** void
- **Side effects:** Sets `bs->ltgtype = LTG_GETFLAG`, updates `decisionmaker`, `ordered`, timing fields; calls `BotGetAlternateRouteGoal` in GT_CTF; calls `BotSetTeamStatus`, `BotRememberLastOrderedTask`.
- **Calls:** `FloatTime`, `BotOppositeTeam`, `BotGetAlternateRouteGoal`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`
- **Notes:** Returns early if flag area numbers are not set up; guarded by `#ifdef MISSIONPACK` for GT_1FCTF.

### BotVoiceChat_Offense
- **Signature:** `void BotVoiceChat_Offense(bot_state_t *bs, int client, int mode)`
- **Purpose:** Orders bot to play offensively; delegates to `BotVoiceChat_GetFlag` in CTF, sets `LTG_HARVEST` in GT_HARVESTER, or `LTG_ATTACKENEMYBASE` otherwise.
- **Calls:** `BotVoiceChat_GetFlag`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`

### BotVoiceChat_Defend
- **Signature:** `void BotVoiceChat_Defend(bot_state_t *bs, int client, int mode)`
- **Purpose:** Orders bot to defend a key area (own flag in CTF, own obelisk in Obelisk/Harvester).
- **Side effects:** `memcpy` copies appropriate global goal into `bs->teamgoal`; sets `LTG_DEFENDKEYAREA`.
- **Calls:** `BotTeam`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`
- **Notes:** Returns early for unsupported gametypes or unknown team.

### BotVoiceChat_Camp
- **Signature:** `void BotVoiceChat_Camp(bot_state_t *bs, int client, int mode)`
- **Purpose:** Orders bot to camp at the commanding client's current location.
- **Inputs:** Uses `BotEntityInfo` to locate the issuing client.
- **Side effects:** Sets `LTG_CAMPORDER`; if client is not found in PVS, sends a "whereareyou" chat and returns.
- **Calls:** `BotEntityInfo`, `BotPointAreaNum`, `VectorCopy`, `VectorSet`, `BotAI_BotInitialChat`, `trap_BotEnterChat`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`

### BotVoiceChat_FollowMe
- **Signature:** `void BotVoiceChat_FollowMe(bot_state_t *bs, int client, int mode)`
- **Purpose:** Orders bot to accompany the issuing client.
- **Side effects:** Sets `LTG_TEAMACCOMPANY`, `bs->formation_dist = 3.5 * 32`; aborts with "whereareyou" if client not in PVS.
- **Calls:** `BotEntityInfo`, `BotPointAreaNum`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`

### BotVoiceChat_ReturnFlag
- **Signature:** `void BotVoiceChat_ReturnFlag(bot_state_t *bs, int client, int mode)`
- **Purpose:** Orders bot to return the stolen flag (CTF/1FCTF only).
- **Side effects:** Sets `LTG_RETURNFLAG`.

### BotVoiceChat_StartLeader / BotVoiceChat_StopLeader
- Brief: Assign or revoke the team leader role by name comparison; `StopLeader` also sets `notleader[client] = qtrue`.

### BotVoiceChat_WhoIsLeader
- Brief: If the bot itself is the team leader, broadcasts "iamteamleader" to team chat and emits `VOICECHAT_STARTLEADER`.

### BotVoiceChat_WantOnDefense / BotVoiceChat_WantOnOffense
- Brief: Toggle `TEAMTP_DEFENDER`/`TEAMTP_ATTACKER` bits in teammate task preference; acknowledge with chat and `ACTION_AFFIRMATIVE`.

### BotVoiceChatCommand
- **Signature:** `int BotVoiceChatCommand(bot_state_t *bs, int mode, char *voiceChat)`
- **Purpose:** Entry point; parses the raw voice chat string, extracts `voiceOnly`, `clientNum`, `color`, and the command token, then dispatches to the matching `voiceCommands[]` handler.
- **Inputs:** Bot state, SAY mode, raw voiceChat string.
- **Outputs/Return:** `qtrue` if a handler was found and executed, `qfalse` otherwise.
- **Side effects:** Delegates all side effects to handler functions.
- **Calls:** `TeamPlayIsOn`, `Q_strncpyz`, `atoi`, `BotSameTeam`, `Q_stricmp`, handler via function pointer.
- **Notes:** Immediately returns `qfalse` for `SAY_ALL` mode (ignores global voice chats). Ignores commands from clients not on the same team.

## Control Flow Notes
This file is not part of the per-frame think loop directly. `BotVoiceChatCommand` is called from the bot's server-command processing path (likely `ai_cmd.c`) when the bot receives a voice chat console message. All handlers mutate `bot_state_t` fields that are consumed during the next AI think frame by the goal/LTG evaluation logic in `ai_dmnet.c` / `ai_team.c`.

## External Dependencies
- `g_local.h` — `bot_state_t`, game globals (`gametype`, `ctf_redflag`, etc.), trap functions
- `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h` — helper functions (`BotSetTeamStatus`, `BotRememberLastOrderedTask`, `BotTeamFlagCarrier`, `BotGetAlternateRouteGoal`, `BotSameTeam`, `BotTeam`, etc.)
- `be_aas.h` — `aas_entityinfo_t`, `BotPointAreaNum`, `BotEntityInfo`
- `be_ai_chat.h`, `be_ea.h` — chat/action emission
- `ui/menudef.h` — `VOICECHAT_*` string constants
- `match.h`, `inv.h`, `syn.h`, `chars.h` — bot AI data constants
- **Defined elsewhere:** `notleader[]` array, goal globals (`ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk`), all `LTG_*` / `TEAM_*_TIME` constants, `FloatTime`, `random`
