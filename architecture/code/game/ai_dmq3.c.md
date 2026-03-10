# code/game/ai_dmq3.c

## File Purpose
Core Quake III Arena bot deathmatch AI implementation. It handles per-frame bot decision-making, enemy detection, combat behavior, team goal selection across all multiplayer gametypes (DM, CTF, 1FCTF, Obelisk, Harvester), obstacle avoidance, and game event processing.

## Core Responsibilities
- Per-frame bot AI tick (`BotDeathmatchAI`) that drives the AI node state machine
- Team goal selection: flag capture, base defense, escort, rush-base, harvest, obelisk attack
- Enemy detection and visibility testing with fog/water attenuation
- Aim prediction (linear and physics-based) and attack decision gating
- Inventory and battle inventory updates from `playerState_t`
- Dynamic obstacle detection and BSP entity activation (buttons, doors, trigger_multiples)
- Game event processing (obituaries, flag status changes, grenade/proxmine avoidance)
- Waypoint pool management and alternative route goal setup

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bot_waypoint_t` | struct (defined elsewhere) | Named waypoint with a `bot_goal_t` and linked-list pointers |
| `bot_activategoal_t` | struct (defined elsewhere) | Describes a BSP entity the bot must reach/shoot to unblock its path |
| `aas_entityinfo_t` | struct | AAS snapshot of an entity (origin, flags, powerups, velocity) |
| `aas_altroutegoal_t` | struct | Pre-computed alternative route waypoint for CTF/team modes |
| `weaponinfo_t` | struct (defined elsewhere) | Weapon speed, spread, projectile damage type |
| `bot_moveresult_t` | struct (defined elsewhere) | Result of a movement attempt, including blocked/failure flags |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `gametype` | `int` | global | Cached game type (GT_CTF, GT_1FCTF, etc.) set at setup |
| `maxclients` | `int` | global | Cached max client count |
| `botai_waypoints[MAX_WAYPOINTS]` | `bot_waypoint_t[128]` | global | Static waypoint pool |
| `botai_freewaypoints` | `bot_waypoint_t *` | global | Head of free-waypoint linked list |
| `lastteleport_origin` / `lastteleport_time` | `vec3_t` / `float` | global | Last teleport event; used to suppress false enemy detection |
| `max_bspmodelindex` | `int` | global | Highest BSP model index; used to classify blocking entities |
| `ctf_redflag`, `ctf_blueflag` | `bot_goal_t` | global | Pre-cached CTF flag goal locations |
| `ctf_neutralflag`, `redobelisk`, `blueobelisk`, `neutralobelisk` | `bot_goal_t` | global (MISSIONPACK) | Pre-cached MissionPack objective locations |
| `red_altroutegoals`, `blue_altroutegoals` | `aas_altroutegoal_t[32]` | global | Alternative route goal arrays per team |
| `altroutegoals_setup` | `int` | global | One-time initialization guard for alt route goals |
| `bot_grapple`…`g_spSkill` | `vmCvar_t` | global | Registered bot behavior cvars |
| `VEC_UP`, `MOVEDIR_UP`, `VEC_DOWN`, `MOVEDIR_DOWN` | `vec3_t` | static | Direction constants for `BotSetMovedir` |

## Key Functions / Methods

### BotDeathmatchAI
- **Signature:** `void BotDeathmatchAI(bot_state_t *bs, float thinktime)`
- **Purpose:** Main per-frame AI entry point. Runs setup on first frames, updates inventory/snapshot/events, runs team AI, then executes up to `MAX_NODESWITCHES` AI node functions.
- **Inputs:** `bs` — bot state; `thinktime` — seconds since last frame
- **Outputs/Return:** void; modifies `bs` extensively
- **Side effects:** Calls `trap_EA_*` to queue input actions; may call `AIEnter_*` to switch AI nodes
- **Calls:** `BotUpdateInventory`, `BotCheckSnapshot`, `BotCheckAir`, `BotCheckConsoleMessages`, `BotTeamAI`, `AIEnter_Seek_LTG`, `BotChat_EnterGame`, `BotResetNodeSwitches`, `bs->ainode(bs)`
- **Notes:** `setupcount` throttles first-frame initialization over multiple frames. Node loop terminates early or logs an error if it cycles ≥ `MAX_NODESWITCHES`.

### BotCTFSeekGoals
- **Signature:** `void BotCTFSeekGoals(bot_state_t *bs)`
- **Purpose:** Selects long-term team goals in CTF based on flag status: rush base (carrying flag), escort carrier, return or capture flag, defend base, or roam.
- **Inputs:** `bs`
- **Outputs/Return:** void; sets `bs->ltgtype`, `bs->teamgoal`, `bs->teamgoal_time`
- **Side effects:** Calls `BotVoiceChat`, `BotSetUserInfo`, `BotGetAlternateRouteGoal`
- **Calls:** `BotCTFCarryingFlag`, `BotTeam`, `BotTeamFlagCarrierVisible`, `BotEnemyFlagCarrierVisible`, `BotRefuseOrder`, `BotAggression`, `BotSetLastOrderedTask`, `BotSetTeamStatus`
- **Notes:** Flag status encoded as a 2-bit integer (`redflagstatus*2 + blueflagstatus` or vice versa).

### BotFindEnemy
- **Signature:** `int BotFindEnemy(bot_state_t *bs, int curenemy)`
- **Purpose:** Scans all clients to find the best enemy target, factoring in visibility (with fog), FOV expansion when health decreased or enemy is shooting, distance, invisibility, and retreat willingness.
- **Inputs:** `bs`; `curenemy` — current enemy index or -1
- **Outputs/Return:** `qtrue` if a new enemy was found; sets `bs->enemy`
- **Side effects:** Calls `BotUpdateBattleInventory`; writes `bs->enemy`, `bs->enemysight_time`, etc.
- **Calls:** `BotEntityInfo`, `EntityIsDead`, `EntityIsInvisible`, `EntityIsShooting`, `EntityCarriesFlag`, `BotEntityVisible`, `BotSameTeam`, `BotWantsToRetreat`
- **Notes:** In GT_OBELISK mode the enemy obelisk is checked first via line-of-sight trace.

### BotEntityVisible
- **Signature:** `float BotEntityVisible(int viewer, vec3_t eye, vec3_t viewangles, float fov, int ent)`
- **Purpose:** Returns a visibility factor [0,1] accounting for BSP occlusion, water surfaces, and fog distance.
- **Inputs:** viewer entity num, eye position, view angles, FOV cone, target entity num
- **Outputs/Return:** float visibility (0 = not visible, ≥ 0.95 = fully visible)
- **Side effects:** Issues up to 6 `BotAI_Trace` calls per entity (3 test points × potential water re-trace)
- **Calls:** `BotEntityInfo`, `InFieldOfVision`, `BotAI_Trace`, `trap_AAS_PointContents`
- **Notes:** Checks entity origin, bottom, and top of bounding box to improve accuracy.

### BotAimAtEnemy
- **Signature:** `void BotAimAtEnemy(bot_state_t *bs)`
- **Purpose:** Computes `bs->ideal_viewangles` toward the enemy, with lead prediction (linear or physics), ground-splash targeting, and aim-accuracy noise injection.
- **Inputs/Outputs:** Reads/writes `bs` fields; no return value
- **Side effects:** Writes `bs->ideal_viewangles`, `bs->aimtarget`; may call `trap_EA_View` for challenge-mode bots
- **Calls:** `BotEntityInfo`, `trap_BotGetWeaponInfo`, `trap_Characteristic_BFloat`, `trap_AAS_PredictClientMovement`, `BotEntityVisible`, `BotAI_Trace`, `vectoangles`
- **Notes:** Aim-skill > 0.8 triggers full AAS movement prediction; < 0.4 does no prediction.

### BotCheckAttack
- **Signature:** `void BotCheckAttack(bot_state_t *bs)`
- **Purpose:** Gates `trap_EA_Attack` based on reaction time, weapon-change cooldown, fire throttle, aim FOV alignment, line-of-sight, and teammate-friendly-fire checks.
- **Inputs:** `bs`
- **Side effects:** Calls `trap_EA_Attack`; sets `bs->flags ^= BFL_ATTACKED`
- **Calls:** `BotEntityInfo`, `trap_Characteristic_BFloat`, `trap_BotGetWeaponInfo`, `BotAI_Trace`, `BotSameTeam`, `InFieldOfVision`
- **Notes:** `WFL_FIRERELEASED` weapons require the attack button to be released before re-firing.

### BotAIBlocked
- **Signature:** `void BotAIBlocked(bot_state_t *bs, bot_moveresult_t *moveresult, int activate)`
- **Purpose:** Handles a blocked movement result: attempts BSP entity activation (buttons/doors/triggers) or falls back to sideward/crouch evasion maneuvers.
- **Inputs:** `bs`; `moveresult` — movement result with block info; `activate` — whether to attempt entity activation
- **Side effects:** May call `BotGoForActivateGoal`, `AIEnter_Seek_ActivateEntity`; resets `bs->ltg_time`/`bs->nbg_time` on prolonged block
- **Calls:** `BotEntityInfo`, `BotGetActivateGoal`, `BotIsGoingToActivateEntity`, `BotGoForActivateGoal`, `BotEnableActivateGoalAreas`, `BotRandomMove`, `trap_BotMoveInDirection`

### BotSetupDeathmatchAI
- **Signature:** `void BotSetupDeathmatchAI(void)`
- **Purpose:** One-time initialization: caches `gametype`/`maxclients`, registers cvars, fetches flag/obelisk goal positions, computes `max_bspmodelindex`, and initializes the waypoint pool.
- **Side effects:** Writes all module-level globals; calls `BotInitWaypoints`
- **Notes:** Called once per map load via `BotAILoadMap`.

### BotUpdateInventory
- **Signature:** `void BotUpdateInventory(bot_state_t *bs)`
- **Purpose:** Mirrors `bs->cur_ps` weapon, ammo, powerup, and holdable item data into `bs->inventory[]` for botlib fuzzy-logic queries.
- **Side effects:** Calls `BotCheckItemPickup` with old inventory snapshot; triggers voice chat/preference changes on new item pickup (MISSIONPACK).

### Notes on minor helpers
- `BotCTFCarryingFlag`, `BotTeam`, `BotOppositeTeam`, `BotEnemyFlag`, `BotTeamFlag` — one-liner accessors for team/flag state.
- `EntityIsDead`, `EntityCarriesFlag`, `EntityIsInvisible`, `EntityIsShooting`, `EntityIsChatting`, `EntityHasQuad` — predicate helpers over `aas_entityinfo_t`.
- `BotCreateWayPoint`, `BotFindWayPoint`, `BotFreeWaypoints`, `BotInitWaypoints` — manage the static waypoint pool.
- `ClientName`, `ClientSkin`, `ClientFromName`, `EasyClientName` — client config-string name utilities.
- `BotAggression`, `BotFeelingBad` — return numeric combat-readiness scores based on inventory.

## Control Flow Notes
- **Init:** `BotSetupDeathmatchAI` is called at map load. `BotDeathmatchAI` defers further per-bot setup via `bs->setupcount` over the first few frames.
- **Per-frame:** `BotDeathmatchAI` is the frame entry point, called from `BotAIStartFrame` (in `ai_main.c`) for each active bot. It calls `BotTeamAI` (team goal state) then iterates `bs->ainode` (a function pointer — Seek_LTG, Seek_NBG, Battle, Stand, etc.) up to `MAX_NODESWITCHES` times per frame.
- **Shutdown:** `BotShutdownDeathmatchAI` resets the `altroutegoals_setup` flag so the next map load reinitializes route goals.

## External Dependencies
- `g_local.h` — `gentity_t`, `level`, `g_entities[]`, `G_ModelIndex`, game trap functions
- `botlib.h` / `be_aas.h` / `be_ea.h` / `be_ai_*.h` — botlib AAS, EA, and AI API
- `ai_main.h` — `bot_state_t`, `BotAI_Print`, `BotAI_Trace`, `BotAI_GetEntityState`, `FloatTime`, `NumBots`, `AINode_*` enums, `AIEnter_*` functions
- `ai_dmnet.h` — `BotTeamAI`, `BotTeamLeader`, `AIEnter_Seek_LTG`, `AIEnter_Stand`, `AIEnter_Seek_ActivateEntity`, `BotValidChatPosition`, node switch utilities
- `ai_chat.h` / `ai_cmd.h` / `ai_team.h` — `BotVoiceChat`, `BotChat_EnterGame`, `BotMatchMessage`, `BotChatTime`, `BotSameTeam` (re-exported here), `BotSetTeamStatus`
- `chars.h`, `inv.h`, `syn.h`, `match.h` — characteristic indices, inventory indices, synonym/match contexts
- `ui/menudef.h` — voice chat string constants
- **Defined elsewhere (called but not defined here):** `BotEntityInfo`, `BotAI_GetClientState`, `BotAI_GetSnapshotEntity`, `BotVisibleTeamMatesAndEnemies` (partially defined here but also referenced by external callers), `trap_AAS_*`, `trap_EA_*`, `trap_Bot*`, `trap_Characteristic_*`
