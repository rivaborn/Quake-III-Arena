# code/game/ai_dmnet.c

## File Purpose
Implements the bot AI finite-state machine (FSM) node system for deathmatch and team-game modes. Each `AINode_*` function is a discrete AI state (seek, battle, respawn, etc.) executed per-frame, and each `AIEnter_*` function transitions the bot into a new state. Also manages long-term goal (LTG) selection and navigation for all supported game types.

## Core Responsibilities
- Defines and drives the bot FSM: Intermission, Observer, Stand, Respawn, Seek_LTG, Seek_NBG, Seek_ActivateEntity, Battle_Fight, Battle_Chase, Battle_Retreat, Battle_NBG
- Selects and tracks long-term goals (LTG) based on game type (DM, CTF, 1FCTF, Obelisk, Harvester) and team strategy (defend, patrol, camp, escort, kill)
- Selects nearby goals (NBG) as short interruptions during LTG navigation
- Handles water/air survival logic via `BotGoForAir` / `BotGetAirGoal`
- Tracks node switches for AI debugging via `BotRecordNodeSwitch` / `BotDumpNodeSwitches`
- Detects and deactivates path obstacles (proximity mines, kamikaze bodies) via `BotClearPath`
- Selects a usable weapon for activation tasks via `BotSelectActivateWeapon`

## Key Types / Data Structures
None defined in this file; all types are from included headers.

| Name | Kind | Purpose |
|---|---|---|
| `bot_state_t` | struct (extern) | Full per-bot state; defined in `ai_main.h` |
| `bot_goal_t` | struct (extern) | Describes a navigation goal (origin, areanum, flags, etc.) |
| `bot_moveresult_t` | struct (extern) | Result from `trap_BotMoveToGoal` including failure flags and view/weapon overrides |
| `aas_entityinfo_t` | struct (extern) | AAS entity snapshot used to track teammates/enemies |
| `bsp_trace_t` | struct (extern) | BSP raycast result |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `numnodeswitches` | `int` | global | Count of recorded AI node switches this frame |
| `nodeswitch` | `char[MAX_NODESWITCHES+1][144]` | global | Ring-buffer of node-switch log strings for debug dumps |

## Key Functions / Methods

### BotResetNodeSwitches
- **Signature:** `void BotResetNodeSwitches(void)`
- **Purpose:** Clears the node-switch counter at the start of each AI frame.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets `numnodeswitches` to 0.
- **Calls:** None
- **Notes:** Called once per frame before processing all bots.

### BotDumpNodeSwitches
- **Signature:** `void BotDumpNodeSwitches(bot_state_t *bs)`
- **Purpose:** Prints all recorded node switches and triggers a fatal error when a bot switches nodes too many times in one frame.
- **Side effects:** Calls `BotAI_Print(PRT_FATAL, ...)` — terminates/asserts.

### BotGetAirGoal
- **Signature:** `int BotGetAirGoal(bot_state_t *bs, bot_goal_t *goal)`
- **Purpose:** Constructs a synthetic goal just at the water surface above the bot so it can surface for air.
- **Inputs:** `bs` — bot state; `goal` — output goal struct
- **Outputs/Return:** `qtrue` if a valid air goal was found, `qfalse` otherwise.
- **Side effects:** Writes `*goal`; performs two BSP traces.
- **Calls:** `BotAI_Trace`, `BotPointAreaNum`

### BotGoForAir
- **Signature:** `int BotGoForAir(bot_state_t *bs, int tfl, bot_goal_t *ltg, float range)`
- **Purpose:** Pushes an air goal onto the goal stack if the bot has been submerged for >6 seconds.
- **Outputs/Return:** `qtrue` if an air goal was pushed.
- **Calls:** `BotGetAirGoal`, `trap_BotPushGoal`, `trap_BotChooseNBGItem`, `trap_AAS_PointContents`, `trap_BotPopGoal`, `trap_BotResetAvoidGoals`

### BotGetItemLongTermGoal
- **Signature:** `int BotGetItemLongTermGoal(bot_state_t *bs, int tfl, bot_goal_t *goal)`
- **Purpose:** Manages the vanilla item-collection LTG: pops a completed goal, calls `trap_BotChooseLTGItem` for a new one, resets avoidance on failure.
- **Outputs/Return:** `qtrue` if a valid LTG exists on the stack.
- **Calls:** `trap_BotGetTopGoal`, `BotReachedGoal`, `BotChooseWeapon`, `trap_BotPopGoal`, `trap_BotChooseLTGItem`, `trap_BotResetAvoidGoals`, `trap_BotResetAvoidReach`

### BotGetLongTermGoal
- **Signature:** `int BotGetLongTermGoal(bot_state_t *bs, int tfl, int retreat, bot_goal_t *goal)`
- **Purpose:** Dispatches to the correct LTG logic based on `bs->ltgtype` and current `gametype` (team-help, accompany, defend, kill, get-item, camp, patrol, CTF/1FCTF/Obelisk/Harvester flags). Falls through to `BotGetItemLongTermGoal` if no type matches.
- **Outputs/Return:** `qtrue` if bot should keep moving toward the returned `*goal`, `qfalse` to halt/stand.
- **Side effects:** May trigger chat, voice chat, crouching, gestures, team goal updates; modifies `bs->ltgtype`, `bs->arrive_time`, etc.
- **Calls:** Extensive — `BotEntityInfo`, `BotEntityVisible`, `BotAI_BotInitialChat`, `trap_BotEnterChat`, `BotVoiceChatOnly`, `trap_EA_Action/Gesture/Crouch`, `trap_AAS_AreaTravelTimeToGoalArea`, `BotAlternateRoute`, `BotGoHarvest`, `BotGetItemLongTermGoal`, etc.

### BotLongTermGoal
- **Signature:** `int BotLongTermGoal(bot_state_t *bs, int tfl, int retreat, bot_goal_t *goal)`
- **Purpose:** Wraps `BotGetLongTermGoal`; additionally handles the "lead teammate" mode where the bot guides a human player.
- **Calls:** `BotGetLongTermGoal`, `BotAI_BotInitialChat`, `trap_BotEnterChat`, `BotEntityInfo`, `BotEntityVisible`

### AIEnter_* / AINode_* (FSM nodes)
Each pair follows the pattern: `AIEnter_X` records the transition and sets `bs->ainode = AINode_X`; `AINode_X` runs every frame.

| Node | Description |
|---|---|
| `AINode_Intermission` | Waits for intermission end; transitions to Stand |
| `AINode_Observer` | Waits while in observer mode |
| `AINode_Stand` | Bot stands still (post-kill chat, post-spawn chat); transitions to Battle_Fight or Seek_LTG |
| `AINode_Respawn` | Delays respawn, fires `trap_EA_Respawn`, transitions to Seek_LTG |
| `AINode_Seek_LTG` | Main roaming state: picks LTG, checks for NBG, moves, reacts to enemies |
| `AINode_Seek_NBG` | Short detour to pick up a nearby item; returns to Seek_LTG on timeout |
| `AINode_Seek_ActivateEntity` | Navigates to/shoots a button/trigger; pops activate goal stack |
| `AINode_Battle_Fight` | Engages visible enemy: attack moves, aim, weapon choice, retreat check |
| `AINode_Battle_Chase` | Chases a recently-lost enemy to their last known position |
| `AINode_Battle_Retreat` | Retreats while still attacking; transitions to suicidal fight if no exit |
| `AINode_Battle_NBG` | Picks up nearby item while maintaining combat awareness |

### BotClearPath
- **Signature:** `void BotClearPath(bot_state_t *bs, bot_moveresult_t *moveresult)`
- **Purpose:** Detects kamikaze bodies and proximity mines blocking the bot's path; aims and shoots at them using splash-damage weapons.
- **Side effects:** Sets `moveresult->weapon`, `moveresult->flags`, calls `trap_EA_Attack`.
- **Calls:** `BotAI_GetEntityState`, `BotAI_Trace`, `trap_EA_Attack`, `InFieldOfVision`

### BotSelectActivateWeapon
- **Signature:** `int BotSelectActivateWeapon(bot_state_t *bs)`
- **Purpose:** Returns the index of the first weapon the bot currently has ammo for, prioritizing MG → Shotgun → Plasma → Lightning (→ Chaingun/Nailgun in MissionPack) → Rail → RL → BFG.
- **Outputs/Return:** Weapon index, or `-1` if none available.

## Control Flow Notes
- Called from `ai_main.c`'s `BotAIStartFrame` each server frame per bot.
- The pattern is: `bs->ainode` holds a function pointer; the frame runner calls it. Each node returns `qtrue` to stay in the node or `qfalse` after transitioning via `AIEnter_*`.
- `BotResetNodeSwitches` is called before all bots; `BotDumpNodeSwitches` is called if `numnodeswitches > MAX_NODESWITCHES`.
- `AINode_Seek_LTG` is the "home" state; all other nodes eventually return here when idle.

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_team.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`, `ui/menudef.h`
- **Defined elsewhere:**
  - `bot_state_t`, `BotResetState`, `BotChat_*`, `BotFindEnemy`, `BotWantsToRetreat/Chase`, `BotAIPredictObstacles`, `BotAIBlocked`, `BotSetupForMovement`, `BotAttackMove`, `BotAimAtEnemy`, `BotCheckAttack`, `BotChooseWeapon`, `BotUpdateBattleInventory`, `BotBattleUseItems`, `BotMapScripts`, `BotTeamGoals`, `BotWantsToCamp`, `BotRoamGoal`, `BotAlternateRoute`, `BotGoHarvest` — all in companion `ai_*.c` files
  - `gametype`, `ctf_redflag`, `ctf_blueflag`, `ctf_neutralflag`, `redobelisk`, `blueobelisk`, `neutralobelisk` — game-mode globals from `ai_team.c` / `ai_dmq3.c`
  - All `trap_*` functions — game-module syscall stubs in `g_syscalls.c`
