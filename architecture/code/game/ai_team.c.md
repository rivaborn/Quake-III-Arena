# code/game/ai_team.c

## File Purpose
Implements the bot team AI leadership system for Quake III Arena, responsible for issuing tactical orders to teammates based on game mode (Team DM, CTF, 1FCTF, Obelisk, Harvester). A single bot acts as team leader and periodically distributes role assignments (defend/attack/escort) to teammates sorted by proximity to the base.

## Core Responsibilities
- Validate and elect a team leader (human or bot)
- Count teammates and sort them by AAS travel time to the team's home base/obelisk
- Re-sort teammates by stored task preferences (defender/attacker/roamer)
- Issue context-sensitive orders per game mode and flag/objective status
- Deliver orders via team chat messages and/or voice chat commands (MISSIONPACK)
- Periodically re-evaluate strategy (randomly toggle aggressive/passive CTF strategy)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_ctftaskpreference_t` | struct | Stores a client's name and defender/attacker preference flag for CTF role assignment |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `ctftaskpreferences` | `bot_ctftaskpreference_t[MAX_CLIENTS]` | global | Per-client CTF task preference cache, indexed by client number; name field used to detect stale entries |

## Key Functions / Methods

### BotValidTeamLeader
- **Signature:** `int BotValidTeamLeader(bot_state_t *bs)`
- **Purpose:** Checks whether the bot's stored team leader name is non-empty and maps to an active client.
- **Inputs:** Bot state pointer
- **Outputs/Return:** `qtrue` if leader is valid, `qfalse` otherwise
- **Side effects:** None
- **Calls:** `ClientFromName`
- **Notes:** Does not validate that the leader is on the same team.

### BotNumTeamMates
- **Signature:** `int BotNumTeamMates(bot_state_t *bs)`
- **Purpose:** Counts active, non-spectator teammates on the bot's team.
- **Inputs:** Bot state pointer
- **Outputs/Return:** Count of same-team, non-spectator players
- **Side effects:** Caches `sv_maxclients` in a static local on first call
- **Calls:** `trap_GetConfigstring`, `Info_ValueForKey`, `BotSameTeam`

### BotClientTravelTimeToGoal
- **Signature:** `int BotClientTravelTimeToGoal(int client, bot_goal_t *goal)`
- **Purpose:** Queries AAS for travel time from a client's current area to a given goal area.
- **Inputs:** Client index, goal pointer
- **Outputs/Return:** Travel time in AAS units; returns `1` if origin area is invalid (zero)
- **Side effects:** None
- **Calls:** `BotAI_GetClientState`, `BotPointAreaNum`, `trap_AAS_AreaTravelTimeToGoalArea`

### BotSortTeamMatesByBaseTravelTime
- **Signature:** `int BotSortTeamMatesByBaseTravelTime(bot_state_t *bs, int *teammates, int maxteammates)`
- **Purpose:** Builds an insertion-sorted list of same-team client indices, ascending by AAS travel time to the team's home flag or obelisk.
- **Inputs:** Bot state, output array, array capacity
- **Outputs/Return:** Number of teammates found; fills `teammates[]`
- **Side effects:** None
- **Calls:** `BotTeam`, `BotClientTravelTimeToGoal`, `BotSameTeam`, `trap_GetConfigstring`
- **Notes:** Goal is selected from `ctf_redflag`/`ctf_blueflag` globals (or `redobelisk`/`blueobelisk` under MISSIONPACK).

### BotSetTeamMateTaskPreference / BotGetTeamMateTaskPreference
- **Purpose:** Write/read a client's CTF role preference into `ctftaskpreferences[]`. `Get` validates by re-checking the stored name against the current client name to detect client slot reuse.
- **Calls:** `ClientName`

### BotSortTeamMatesByTaskPreference
- **Signature:** `int BotSortTeamMatesByTaskPreference(bot_state_t *bs, int *teammates, int numteammates)`
- **Purpose:** Reorders a teammate array so defenders come first, then roamers, then attackers.
- **Inputs:** Bot state, in/out teammate array, count
- **Outputs/Return:** Returns unchanged count; array is reordered in-place via three temporary sub-arrays
- **Side effects:** None

### BotSayTeamOrderAlways / BotSayTeamOrder / BotSayVoiceTeamOrder
- **Purpose:** Deliver a queued chat message or voice command to a specific client or broadcast to team. `BotSayTeamOrder` is a no-op for text in MISSIONPACK builds (voice-only). `BotSayTeamOrderAlways` routes self-addressed orders to the console queue instead of chat.
- **Calls:** `trap_BotGetChatMessage`, `trap_BotEnterChat`, `trap_BotQueueConsoleMessage`, `trap_EA_Command`

### BotCTFOrders
- **Signature:** `void BotCTFOrders(bot_state_t *bs)`
- **Purpose:** Dispatcher that encodes the two-flag status into a 0–3 integer and delegates to one of four sub-functions (`BotCTFOrders_BothFlagsAtBase`, `_EnemyFlagNotAtBase`, `_FlagNotAtBase`, `_BothFlagsNotAtBase`).
- **Side effects:** Sends team chat/voice orders

### BotTeamAI
- **Signature:** `void BotTeamAI(bot_state_t *bs)`
- **Purpose:** Per-frame entry point for team AI. Validates/elects a team leader, then, only when this bot is the leader, checks whether conditions (team size change, flag status change, timer) warrant re-issuing orders for the current game mode.
- **Inputs:** Bot state
- **Side effects:** Modifies `bs->teamleader`, `bs->teamgiveorders_time`, `bs->numteammates`, `bs->ctfstrategy`, `bs->forceorders`, `bs->flagstatuschanged`; sends chat messages
- **Calls:** `BotValidTeamLeader`, `FindHumanTeamLeader`, `BotNumTeamMates`, `BotCTFOrders`, `BotTeamOrders`, `Bot1FCTFOrders`, `BotObeliskOrders`, `BotHarvesterOrders`
- **Notes:** Returns early if `gametype < GT_TEAM`. Strategy toggle occurs if no captures for 4 minutes (`random() < 0.4`).

### FindHumanTeamLeader
- **Signature:** `int FindHumanTeamLeader(bot_state_t *bs)`
- **Purpose:** Scans `g_entities` for the first non-bot, non-`notleader` human on the same team and installs them as team leader.
- **Calls:** `BotSameTeam`, `ClientName`, `BotSetLastOrderedTask`, `BotVoiceChat_Defend`

## Control Flow Notes
`BotTeamAI` is called once per bot per frame from the game's AI frame loop (`BotAIStartFrame` → per-bot think). It is purely an output/command-dispatch layer: it reads game state from `bs` and level globals, issues orders via the botlib chat/EA traps, and writes timing fields back to `bs`. No rendering or physics involvement.

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h`, `ai_vcmd.h`, `match.h`, `../../ui/menudef.h`
- **Defined elsewhere:** `ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk` (goal structs from `ai_dmq3.c`/`ai_main.c`); `gametype`, `notleader[]`, `g_entities[]`; `BotSameTeam`, `BotTeam`, `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotPointAreaNum`, `ClientName`, `ClientFromName`, `FloatTime`, `BotSetLastOrderedTask`, `BotVoiceChat_Defend`; all `trap_*` syscalls
