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

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_char.h`, `be_ai_chat.h`, `be_ai_gen.h`, `be_ai_goal.h`, `be_ai_move.h`, `be_ai_weap.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`; conditionally `ui/menudef.h` (MissionPack)
- **Defined elsewhere:** `bot_state_t`, `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotAI_Trace`, `EasyClientName`, `ClientName`, `BotEntityInfo`, `BotEntityVisible`, `BotSameTeam`, `BotIsDead`, `BotIsObserver`, `EntityIsDead`, `EntityIsInvisible`, `EntityIsShooting`, `TeamPlayIsOn`, `FloatTime`, `gametype`, `bot_nochat`, `bot_fastchat`, `g_entities`, all `trap_*` syscalls

# code/game/ai_chat.h
## File Purpose
Public interface header declaring bot AI chat functions for Quake III Arena. It exposes event-driven chat triggers and utility functions that allow bots to send contextually appropriate chat messages during gameplay.

## Core Responsibilities
- Declare chat event hooks for game lifecycle events (enter/exit game, level start/end)
- Declare combat-contextual chat triggers (hit, death, kill, suicide)
- Declare utility functions for chat timing, position validation, and testing

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` / `g_local.h` (defined elsewhere)
- Implementation resides in `code/game/ai_chat.c`

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

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_char/chat/gen/goal/move/weap.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`, `ui/menudef.h`
- **Defined elsewhere (used here):** `bot_state_t`, `BotSetTeamStatus`, `BotRememberLastOrderedTask`, `BotGetAlternateRouteGoal`, `BotOppositeTeam`, `BotSameTeam`, `BotTeam`, `BotFindWayPoint`, `BotCreateWayPoint`, `BotFreeWaypoints`, `BotVoiceChat`, `BotVoiceChatOnly`, `TeamPlayIsOn`, `ClientFromName`, `ClientOnSameTeamFromName`, `EasyClientName`, `BotAI_BotInitialChat`, `BotAI_Trace`, `FloatTime`, `gametype`, `ctf_redflag`, `ctf_blueflag`, all `trap_*` syscalls.

# code/game/ai_cmd.h
## File Purpose
Header file for the bot AI command/message processing subsystem in Quake III Arena. It declares the public interface for bot team-command parsing and team goal reporting used by the game module's AI layer.

## Core Responsibilities
- Exposes the `BotMatchMessage` function for parsing and dispatching incoming chat/voice commands to a bot
- Exposes `BotPrintTeamGoal` for outputting the bot's current team objective
- Declares the `notleader` array used to track which clients have been flagged as non-leaders across the bot subsystem

## External Dependencies
- **`bot_state_t`** — defined in `ai_main.h` or `g_local.h`; the central bot runtime state structure.
- **`MAX_CLIENTS`** — defined in `q_shared.h`; engine-wide client count limit.
- Implementation lives in `code/game/ai_cmd.c`.

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

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_team.h`, `chars.h`, `inv.h`, `syn.h`, `match.h`, `ui/menudef.h`
- **Defined elsewhere:**
  - `bot_state_t`, `BotResetState`, `BotChat_*`, `BotFindEnemy`, `BotWantsToRetreat/Chase`, `BotAIPredictObstacles`, `BotAIBlocked`, `BotSetupForMovement`, `BotAttackMove`, `BotAimAtEnemy`, `BotCheckAttack`, `BotChooseWeapon`, `BotUpdateBattleInventory`, `BotBattleUseItems`, `BotMapScripts`, `BotTeamGoals`, `BotWantsToCamp`, `BotRoamGoal`, `BotAlternateRoute`, `BotGoHarvest` — all in companion `ai_*.c` files
  - `gametype`, `ctf_redflag`, `ctf_blueflag`, `ctf_neutralflag`, `redobelisk`, `blueobelisk`, `neutralobelisk` — game-mode globals from `ai_team.c` / `ai_dmq3.c`
  - All `trap_*` functions — game-module syscall stubs in `g_syscalls.c`

# code/game/ai_dmnet.h
## File Purpose
Public interface header for the Quake III Arena deathmatch bot AI state machine. It declares the state-enter functions (`AIEnter_*`) and state-node functions (`AINode_*`) that implement the bot's high-level behavioral FSM, along with diagnostic utilities.

## Core Responsibilities
- Declare the FSM state-entry transition functions (`AIEnter_*`) called when a bot switches states
- Declare the FSM state-node execution functions (`AINode_*`) called each frame to run the current state's logic
- Export node-switch diagnostic helpers for debugging bot behavior
- Define the `MAX_NODESWITCHES` guard constant to cap FSM transition history

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` (game-side bot state structure)
- `ai_dmnet.c` — provides all implementations declared here
- Consumers: `ai_main.c`, `ai_dmq3.c`, team AI files

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

## External Dependencies
- `g_local.h` — `gentity_t`, `level`, `g_entities[]`, `G_ModelIndex`, game trap functions
- `botlib.h` / `be_aas.h` / `be_ea.h` / `be_ai_*.h` — botlib AAS, EA, and AI API
- `ai_main.h` — `bot_state_t`, `BotAI_Print`, `BotAI_Trace`, `BotAI_GetEntityState`, `FloatTime`, `NumBots`, `AINode_*` enums, `AIEnter_*` functions
- `ai_dmnet.h` — `BotTeamAI`, `BotTeamLeader`, `AIEnter_Seek_LTG`, `AIEnter_Stand`, `AIEnter_Seek_ActivateEntity`, `BotValidChatPosition`, node switch utilities
- `ai_chat.h` / `ai_cmd.h` / `ai_team.h` — `BotVoiceChat`, `BotChat_EnterGame`, `BotMatchMessage`, `BotChatTime`, `BotSameTeam` (re-exported here), `BotSetTeamStatus`
- `chars.h`, `inv.h`, `syn.h`, `match.h` — characteristic indices, inventory indices, synonym/match contexts
- `ui/menudef.h` — voice chat string constants
- **Defined elsewhere (called but not defined here):** `BotEntityInfo`, `BotAI_GetClientState`, `BotAI_GetSnapshotEntity`, `BotVisibleTeamMatesAndEnemies` (partially defined here but also referenced by external callers), `trap_AAS_*`, `trap_EA_*`, `trap_Bot*`, `trap_Characteristic_*`

# code/game/ai_dmq3.h
## File Purpose
Public interface header for Quake III Arena's deathmatch bot AI subsystem. It declares all functions and extern symbols used by `ai_dmq3.c` and consumed by the broader game-side bot framework (primarily `ai_main.c`).

## Core Responsibilities
- Declare the bot deathmatch AI lifecycle functions (setup, shutdown, per-frame think)
- Expose combat decision helpers (enemy detection, weapon selection, aggression, retreat logic)
- Declare movement, inventory, and situational-awareness utilities
- Expose CTF and (conditionally) Mission Pack game-mode goal-setting routines
- Declare waypoint management functions
- Export global game-state variables and cvars used across bot AI files

## External Dependencies
- `bot_state_t`, `bot_waypoint_t`, `bot_goal_t`, `bot_moveresult_t`, `bot_activategoal_t` — defined in `ai_main.h` / `g_local.h`
- `aas_entityinfo_t` — defined in `be_aas.h` / botlib headers
- `vmCvar_t` — defined in `q_shared.h` / `qcommon.h`
- `vec3_t`, `qboolean` — defined in `q_shared.h`
- CTF flag constants (`CTF_FLAG_NONE/RED/BLUE`) and skin macros defined in this file; consumed by `ai_dmq3.c` and CTF-aware callers

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

## External Dependencies

- `g_local.h` / `g_public.h` — game entity types, trap functions, game globals (`g_entities`, `level`, `maxclients`, `gametype`)
- `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h` — botlib API: AAS, elementary actions, chat/goal/move/weapon AI
- `ai_dmq3.h` / `ai_dmnet.h` / `ai_chat.h` / `ai_cmd.h` / `ai_vcmd.h` — higher-level deathmatch AI (`BotDeathmatchAI`, `BotSetupDeathmatchAI`, `BotChat_ExitGame`, etc.)
- `chars.h`, `inv.h`, `syn.h` — bot character, inventory, and synonym constants
- `trap_*` functions — VM syscall interface to the engine (AAS, EA, BotLib, Cvar, Trace, etc.), defined elsewhere in the engine/game syscall layer
- `ExitLevel` — declared extern, defined in `g_main.c`

# code/game/ai_main.h
## File Purpose
Central header for Quake III Arena's in-game bot AI system. Defines the monolithic `bot_state_t` structure that tracks all per-bot runtime state, along with constants for bot behavior flags, long-term goal types, team/CTF strategies, and shared AI utility function declarations.

## Core Responsibilities
- Define all bot behavioral flag constants (`BFL_*`) and long-term goal type constants (`LTG_*`)
- Define goal dedication timeouts for team and CTF scenarios
- Declare `bot_state_t`, the master per-bot state record spanning movement, goals, combat, team, and CTF data
- Declare `bot_waypoint_t` for checkpoint/patrol point linked lists
- Declare `bot_activategoal_t` for a stack of interactive object activation goals
- Expose the `FloatTime()` macro and utility function declarations used across AI subsystems

## External Dependencies
- `bg_public.h` / game headers: `playerState_t`, `usercmd_t`, `entityState_t`, `vec3_t`
- `botlib.h` / `be_aas.h`: `bot_goal_t`, `bot_settings_t`, `aas_entityinfo_t`, `bsp_trace_t`
- `be_ai_move.h`: move state handle type (used in `ms` field)
- Trap functions (`trap_Trace`, `trap_Printf`, etc.) — defined in `g_syscalls.c`, called from game VM

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

## External Dependencies
- **Includes:** `g_local.h`, `botlib.h`, `be_aas.h`, `be_ea.h`, `be_ai_*.h`, `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h`, `ai_vcmd.h`, `match.h`, `../../ui/menudef.h`
- **Defined elsewhere:** `ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk` (goal structs from `ai_dmq3.c`/`ai_main.c`); `gametype`, `notleader[]`, `g_entities[]`; `BotSameTeam`, `BotTeam`, `BotAI_BotInitialChat`, `BotAI_GetClientState`, `BotPointAreaNum`, `ClientName`, `ClientFromName`, `FloatTime`, `BotSetLastOrderedTask`, `BotVoiceChat_Defend`; all `trap_*` syscalls

# code/game/ai_team.h
## File Purpose
Public interface header for Quake III Arena's bot team AI module. Declares the entry points and utility functions used by other game modules to drive team-based bot behavior and voice communication.

## Core Responsibilities
- Expose the main team AI tick function (`BotTeamAI`) for per-frame bot updates
- Provide teammate task preference get/set API for coordinating team roles
- Declare voice chat dispatch functions for bot-to-client voice communication

## External Dependencies
- `bot_state_t` — defined in `ai_main.h` / `g_local.h` (defined elsewhere)
- Implementation bodies reside in `code/game/ai_team.c`

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

## External Dependencies
- `g_local.h` — `bot_state_t`, game globals (`gametype`, `ctf_redflag`, etc.), trap functions
- `ai_main.h`, `ai_dmq3.h`, `ai_chat.h`, `ai_cmd.h`, `ai_dmnet.h`, `ai_team.h` — helper functions (`BotSetTeamStatus`, `BotRememberLastOrderedTask`, `BotTeamFlagCarrier`, `BotGetAlternateRouteGoal`, `BotSameTeam`, `BotTeam`, etc.)
- `be_aas.h` — `aas_entityinfo_t`, `BotPointAreaNum`, `BotEntityInfo`
- `be_ai_chat.h`, `be_ea.h` — chat/action emission
- `ui/menudef.h` — `VOICECHAT_*` string constants
- `match.h`, `inv.h`, `syn.h`, `chars.h` — bot AI data constants
- **Defined elsewhere:** `notleader[]` array, goal globals (`ctf_redflag`, `ctf_blueflag`, `redobelisk`, `blueobelisk`), all `LTG_*` / `TEAM_*_TIME` constants, `FloatTime`, `random`

# code/game/ai_vcmd.h
## File Purpose
Public interface header for bot voice chat command handling in Quake III Arena's game-side AI system. Declares functions used to process and respond to voice chat events as part of bot behavioral logic.

## Core Responsibilities
- Expose the bot voice chat command dispatcher (`BotVoiceChatCommand`)
- Expose the "defend" voice chat response handler (`BotVoiceChat_Defend`)
- Serve as the include boundary between `ai_vcmd.c` and other game AI modules

## External Dependencies
- `bot_state_t` — defined elsewhere, likely `ai_main.h`
- Implementation body: `ai_vcmd.c` (noted in `$Archive` comment)
- No standard library includes in this header

# code/game/be_aas.h
## File Purpose
Public header exposing the Area Awareness System (AAS) interface to the game-side AI layer. It defines travel flags, spatial query result types, and movement prediction structures that bot AI code uses to navigate and reason about the world.

## Core Responsibilities
- Define all `TFL_*` travel type flags used to filter/allow navigation reachabilities
- Declare `aas_trace_t` for AAS-space sweep tests
- Declare `aas_entityinfo_t` for per-entity state visible to bots
- Declare `aas_areainfo_t` for querying area spatial/content metadata
- Define `SE_*` stop-event flags for client movement prediction
- Declare `aas_clientmove_t` for movement simulation results
- Declare `aas_altroutegoal_t` / `aas_predictroute_t` for alternate-route and route-prediction queries

## External Dependencies
- `qboolean`, `vec3_t` — defined in `q_shared.h` (engine shared types)
- `cplane_t` — referenced in the commented-out `bsp_trace_t` block; defined in `q_shared.h`
- `botlib.h` — noted inline as the canonical home for `bsp_trace_t` / `bsp_surface_t` (excluded via comment guard)
- `MAX_STRINGFIELD` — guarded define, may also be provided by botlib headers

# code/game/be_ai_char.h
## File Purpose
Public API header for the bot character system, exposing functions to load, query, and free bot personality/skill profiles. It defines the interface through which game code retrieves typed characteristic values (float, integer, string) from a named character file.

## Core Responsibilities
- Declare the bot character load/free lifecycle functions
- Expose typed accessors for individual bot characteristics by index
- Provide bounded variants of numeric accessors to clamp values within caller-specified ranges
- Declare a global shutdown function to release all cached character data

## External Dependencies
- No includes in this header.
- All function bodies defined in `code/botlib/be_ai_char.c` (defined elsewhere).
- Consumed via the botlib interface layer (`be_interface.c`) or directly by game bot code.

# code/game/be_ai_chat.h
## File Purpose
Declares the public interface for the bot chat AI subsystem, defining data structures and function prototypes used to manage bot console message queues, pattern-based chat matching, and chat message generation/delivery.

## Core Responsibilities
- Define constants for message size limits, gender flags, and chat target types
- Declare the console message linked-list node structure for per-bot message queues
- Declare match variable and match result structures for template-based message parsing
- Expose lifecycle functions for the chat AI subsystem (setup/shutdown, alloc/free state)
- Expose functions for queuing, retrieving, and removing console messages
- Expose functions for selecting, composing, and sending chat replies
- Expose utility functions for string matching, synonym replacement, and whitespace normalization

## External Dependencies
- No includes visible in this header; implementation resides in `botlib/be_ai_chat.c`.
- `MAX_MESSAGE_SIZE`, `MAX_MATCHVARIABLES`, gender/target constants are self-contained in this file.
- All function bodies are **defined elsewhere** (botlib shared library, linked via `botlib_export_t` function table).

# code/game/be_ai_gen.h
## File Purpose
Public header exposing the genetic selection interface used by the bot AI system. It declares a single utility function for selecting parent and child candidates based on a ranked fitness array, supporting evolutionary/genetic algorithm techniques in bot decision-making.

## Core Responsibilities
- Declare the `GeneticParentsAndChildSelection` interface for use by bot AI modules
- Expose genetic selection logic as a callable contract across translation units

## External Dependencies
- No includes in this header.
- Implementation defined elsewhere: `code/botlib/be_ai_gen.c`
- Consumed by: `code/game/` bot AI modules and potentially `code/botlib/` internals

# code/game/be_ai_goal.h
## File Purpose
Public interface header for the bot goal AI subsystem in Quake III Arena's botlib. It defines the `bot_goal_t` structure and declares all functions used to manage bot goals, goal stacks, item weights, and fuzzy logic for goal selection.

## Core Responsibilities
- Define the `bot_goal_t` structure representing a navigable destination
- Declare goal state lifecycle management (alloc, reset, free)
- Declare goal stack push/pop/query operations
- Declare long-term goal (LTG) and nearby goal (NBG) item selection
- Declare item weight loading and fuzzy logic mutation/interbreeding
- Declare level item initialization and dynamic entity item updates
- Declare avoid-goal tracking and timing

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `MAX_AVOIDGOALS`, `MAX_GOALSTACK`, `GFL_*` flags — defined in this file
- All function bodies — defined in `code/botlib/be_ai_goal.c`
- AAS travel flag constants (`travelflags`) — defined in `be_aas.h` / `aasfile.h`

# code/game/be_ai_move.h
## File Purpose
Public header defining the movement AI interface for Quake III's bot library. It declares movement type flags, move state flags, result flags, key data structures, and the full function API used by game code to drive bot locomotion.

## Core Responsibilities
- Define bitmask constants for movement types (walk, crouch, jump, grapple, rocket jump)
- Define bitmask constants for movement state flags (on-ground, swimming, teleported, etc.)
- Define bitmask constants for movement result flags (view override, blocked, obstacle, elevator)
- Declare `bot_initmove_t` for seeding a move state from player/entity state
- Declare `bot_moveresult_t` for communicating locomotion outcomes back to callers
- Declare `bot_avoidspot_t` for spatial hazard avoidance regions
- Expose the full movement AI lifecycle API (alloc/init/move/free)

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `bot_goal_t` — defined in `be_ai_goal.h`
- `bot_initmove_t.or_moveflags` values (`MFL_ONGROUND`, etc.) sourced from engine `playerState_t` by the caller
- Implementation: `code/botlib/be_ai_move.c`

# code/game/be_ai_weap.h
## File Purpose
Public header defining data structures and function prototypes for the bot weapon AI subsystem. It describes projectile and weapon properties used by the botlib to reason about weapon selection and ballistics.

## Core Responsibilities
- Define flags for projectile behavior (window damage, return-to-owner)
- Define flags for weapon firing behavior (key-up fire release)
- Define damage type bitmasks (impact, radial, visible)
- Declare `projectileinfo_t` and `weaponinfo_t` structs used throughout the bot weapon system
- Expose the weapon AI lifecycle API (setup, shutdown, alloc, free, reset)
- Expose weapon selection and information query functions

## External Dependencies
- `MAX_STRINGFIELD` — defined in botlib shared headers (e.g., `botlib.h` or `be_aas.h`)
- `vec3_t` — defined in `q_shared.h`
- All function bodies defined in `code/botlib/be_ai_weap.c`

# code/game/be_ea.h
## File Purpose
Declares the "Elementary Actions" (EA) API for the Quake III bot library. It provides the bot system's lowest-level abstraction over client input, translating high-level bot decisions into discrete client commands and movement/view inputs that are eventually forwarded to the server.

## Core Responsibilities
- Declare client-command EA functions (chat, arbitrary commands, discrete button actions)
- Declare movement EA functions (crouch, walk, strafe, jump, directional move)
- Declare view/weapon EA functions (aim direction, weapon selection)
- Declare input aggregation and dispatch functions (end-of-frame flush, input readback, reset)
- Declare module lifecycle entry points (setup/shutdown)

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `bot_input_t` — defined in `botlib.h` / `be_aas_def.h`
- Implementation: `botlib/be_ea.c`
- Consumed by: `game/ai_move.c`, `game/ai_dmq3.c`, and other game-side AI modules via the botlib interface

# code/game/bg_lib.c
## File Purpose
A self-contained replacement for the standard C library, compiled exclusively for use in Quake III's virtual machine (Q3_VM) target. It provides `qsort`, string functions, math functions, printf-family functions, and numeric parsing so that VM-compiled game modules (game, cgame, ui) do not depend on the host platform's libc.

## Core Responsibilities
- Provide a portable `qsort` (Bentley-McIlroy) usable in both VM and native builds
- Supply string functions (`strlen`, `strcpy`, `strcat`, `strcmp`, `strchr`, `strstr`) for VM builds
- Supply character-classification helpers (`tolower`, `toupper`) for VM builds
- Provide table-driven trigonometry (`sin`, `cos`, `acos`, `atan2`) and `tan` for VM builds
- Implement numeric conversion (`atoi`, `atof`, `_atoi`, `_atof`) with pointer-advance variants
- Implement a minimal `vsprintf`/`sscanf` for formatted I/O inside the VM
- Provide `memmove`, `rand`/`srand`, `abs`, `fabs`

## External Dependencies
- **Includes:** `q_shared.h` (provides `qtrue`, `M_PI`, `size_t`, `va_list`, and the `Q3_VM` macro)
- **Defined elsewhere:** `cmp_t` is conditionally typedef'd here only when `Q3_VM` is not defined; under `Q3_VM` it is assumed provided by `bg_lib.h` (included via `q_shared.h → bg_lib.h`)
- **No heap allocation:** All functions operate on caller-supplied buffers or static/stack storage

# code/game/bg_lib.h
## File Purpose
A self-contained replacement header for standard C library declarations, intended exclusively for use when compiling game code targeting the Quake III virtual machine (QVM). It is explicitly not included in native host builds.

## Core Responsibilities
- Provides `size_t` and `va_list` type definitions for the VM environment
- Declares integer limit macros (`INT_MAX`, `CHAR_BIT`, etc.) normally found in `<limits.h>`
- Declares variadic argument macros (`va_start`, `va_arg`, `va_end`) normally from `<stdarg.h>`
- Declares string manipulation function prototypes replacing `<string.h>`
- Declares memory operation prototypes replacing `<string.h>`/`<memory.h>`
- Declares math function prototypes replacing `<math.h>`
- Declares misc stdlib prototypes (`qsort`, `rand`, `atoi`, `atof`, etc.) replacing `<stdlib.h>`

## External Dependencies
- No includes — this file is itself the bottom of the dependency chain for VM builds.
- All declared symbols are **defined in** `code/game/bg_lib.c` (not inferable from this file alone, but implied by the file comment).
- `va_start`/`va_arg`/`va_end` macros assume a simple cdecl-style stack layout matching the QVM's int-aligned argument passing; they are **not** portable to x86-64 or other ABIs and must never be used in native builds.

# code/game/bg_local.h
## File Purpose
Internal header for the "bg" (both-game) player movement subsystem, shared between the game server and client-side prediction code. It declares the private `pml_t` locals struct, physics tuning constants, and exposes internal pmove helper function signatures that are used across the `bg_pmove.c` and `bg_slidemove.c` translation units.

## Core Responsibilities
- Define movement physics constants (slope limits, step height, jump velocity, timers)
- Declare `pml_t`, the per-frame local movement state that is zeroed before every `Pmove` call
- Expose `pm` and `pml` as extern globals shared across bg source files
- Declare extern movement parameter floats (speed, acceleration, friction tuning values)
- Expose the internal utility function prototypes used only within the bg subsystem

## External Dependencies
- **`q_shared.h` / `bg_public.h`** — `vec3_t`, `trace_t`, `qboolean`, `pmove_t` types (defined elsewhere)
- `pmove_t` — defined in `bg_public.h`
- `vec3_t`, `trace_t` — defined in `q_shared.h`
- All `extern` variables are **defined** in `bg_pmove.c`

# code/game/bg_misc.c
## File Purpose
Defines the master item registry (`bg_itemlist`) for all pickups in Quake III Arena and provides stateless utility functions shared between the server game and client game modules for item lookup, trajectory evaluation, player state conversion, and event management.

## Core Responsibilities
- Declares and initializes the global `bg_itemlist[]` array containing every item definition (weapons, ammo, armor, health, powerups, holdables, team items)
- Provides item lookup functions by powerup tag, holdable tag, weapon tag, and pickup name
- Implements trajectory position and velocity evaluation for all `trType_t` variants
- Determines whether a player can pick up a given item (`BG_CanItemBeGrabbed`) with full gametype/team/MISSIONPACK awareness
- Tests spatial proximity between a player and an item entity
- Manages the predictable event ring-buffer in `playerState_t`
- Converts `playerState_t` → `entityState_t` (both interpolated and extrapolated variants)
- Handles jump-pad velocity application and event generation

## External Dependencies
- `q_shared.h` — core math macros (`VectorCopy`, `VectorMA`, `VectorScale`, `VectorClear`, `SnapVector`), type definitions (`vec3_t`, `playerState_t`, `entityState_t`, `trajectory_t`, `qboolean`), `Com_Error`, `Com_Printf`, `Q_stricmp`, `AngleNormalize180`, `vectoangles`
- `bg_public.h` — `gitem_t`, `itemType_t`, `powerup_t`, `holdable_t`, `weapon_t`, `entity_event_t`, `gametype_t`, `DEFAULT_GRAVITY`, `GIB_HEALTH`, `STAT_*`, `PERS_*`, `PW_*`, `HI_*`, `WP_*`, `EV_*`, `ET_*`, `TR_*`
- `trap_Cvar_VariableStringBuffer` — declared (not defined) in this file; resolved at link time against the VM trap table (cgame or game module)
- `sin`, `cos`, `fabs` — C math library (or VM substitutes via `bg_lib`)

# code/game/bg_pmove.c
## File Purpose
Implements the core player movement (pmove) system for Quake III Arena, shared between the server and client game modules. Takes a `pmove_t` (containing a `playerState_t` and `usercmd_t`) as input and produces an updated `playerState_t` as output. Designed for deterministic client-side prediction.

## Core Responsibilities
- Simulates all player movement modes: walking, air, water, fly, noclip, grapple, dead, spectator
- Applies friction and acceleration per-medium (ground, water, flight, spectator)
- Detects and handles ground contact, slope clamping, and the "all solid" edge case
- Manages jump, crouch/duck, water level, and water jump logic
- Drives weapon state transitions (raising, dropping, firing, ammo consumption)
- Drives legs and torso animation state machines via toggle-bit animation indices
- Generates predictable player events (footsteps, splashes, fall damage, weapon fire, etc.)
- Chops long frames into sub-steps via `Pmove` to prevent framerate-dependent behavior

## External Dependencies

- **Includes:** `q_shared.h`, `bg_public.h`, `bg_local.h`
- **Defined elsewhere:**
  - `PM_SlideMove`, `PM_StepSlideMove` — defined in `bg_slidemove.c`
  - `BG_AddPredictableEventToPlayerstate` — defined in `bg_misc.c`
  - `trap_SnapVector` — syscall stub; platform-specific (snaps float vector components to integers)
  - `AngleVectors`, `VectorNormalize`, `DotProduct`, etc. — `q_shared.c` / `q_math.c`
  - `bg_itemlist` — defined in `bg_misc.c`
  - `Com_Printf` — engine/qcommon

# code/game/bg_public.h
## File Purpose
Shared header defining all game-logic constants, enumerations, and data structures used by both the server-side game module (`game`) and the client-side game module (`cgame`). It establishes the contract between those two VMs and the engine for entity state, player state, items, movement, and events.

## Core Responsibilities
- Define config-string indices (`CS_*`) for server-to-client communication
- Declare all game enumerations: game types, powerups, weapons, holdables, entity types, entity events, animations, means of death
- Define the `pmove_t` context struct and declare the `Pmove` / `PM_UpdateViewAngles` entry points
- Define `player_state` index enumerations (`statIndex_t`, `persEnum_t`)
- Declare the item system (`gitem_t`, `bg_itemlist`, `BG_Find*` helpers)
- Declare shared BG utility functions for trajectory evaluation, event injection, and state conversion
- Define Kamikaze effect timing and sizing constants

## External Dependencies
- `q_shared.h` — `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `vec3_t`, `trace_t`, `qboolean`, `CONTENTS_*`, `MAX_*` constants, `CS_SERVERINFO`/`CS_SYSTEMINFO`
- `MISSIONPACK` preprocessor define — gates additional weapons (`WP_NAILGUN`, `WP_PROX_LAUNCHER`, `WP_CHAINGUN`), powerups, means of death, and entity flags for the Team Arena expansion
- All `BG_*` function bodies defined in `bg_misc.c`, `bg_pmove.c`, `bg_slidemove.c`, `bg_lib.c`

# code/game/bg_slidemove.c
## File Purpose
Implements the sliding collision-response movement for the Quake III pmove system. It resolves player velocity against world geometry by iteratively tracing and clipping velocity along collision planes, and handles automatic step-up over ledges.

## Core Responsibilities
- Trace player movement each frame and clip velocity against hit planes
- Handle up to `MAX_CLIP_PLANES` (5) simultaneous collision planes per move iteration
- Apply gravity interpolation during slide moves
- Detect and resolve two-plane crease collisions via cross-product projection
- Stop the player dead on triple-plane interactions
- Step up over geometry up to `STEPSIZE` (18 units) high via `PM_StepSlideMove`
- Fire step-height events (`EV_STEP_4/8/12/16`) for audio/animation feedback

## External Dependencies
- `q_shared.h` — `vec3_t`, `trace_t`, `qboolean`, vector math macros (`DotProduct`, `VectorMA`, `CrossProduct`, etc.)
- `bg_public.h` — `pmove_t`, `playerState_t`, `EV_STEP_*` event enums, `MAXTOUCH`
- `bg_local.h` — `pml_t`, `STEPSIZE`, `OVERCLIP`, `JUMP_VELOCITY`; extern declarations for `pm`, `pml`, `c_pmove`; declarations of `PM_ClipVelocity`, `PM_AddTouchEnt`, `PM_AddEvent`
- **Defined elsewhere:** `PM_ClipVelocity` (bg_pmove.c), `PM_AddTouchEnt` (bg_pmove.c), `PM_AddEvent` (bg_pmove.c), `pm->trace` callback (set by caller in game/cgame), `Com_Printf` (engine)

# code/game/botlib.h
## File Purpose
Defines the public API boundary between the Quake III game module and the bot AI library (botlib). It declares all function pointer tables (vtables) used to import engine services into botlib and export bot subsystem capabilities back to the game.

## Core Responsibilities
- Define the versioned `botlib_export_t` / `botlib_import_t` interface structs
- Declare input/state types (`bot_input_t`, `bot_entitystate_t`, `bsp_trace_t`) shared across the boundary
- Group bot subsystem exports into nested vtable structs: `aas_export_t`, `ea_export_t`, `ai_export_t`
- Define action flag bitmasks used to encode bot commands
- Define error codes (`BLERR_*`) and print type constants for botlib diagnostics
- Document all configurable library variables and their defaults in a reference comment block

## External Dependencies
- `vec3_t`, `cplane_t`, `qboolean` — defined in `q_shared.h`
- `fileHandle_t`, `fsMode_t` — defined in `q_shared.h` / `qcommon.h`
- `pc_token_t` — defined in the botlib script/precompiler headers (`l_precomp.h`)
- Forward-declared structs (`aas_clientmove_s`, `bot_goal_s`, etc.) — defined in respective `be_aas_*.h` / `be_ai_*.h` headers
- `QDECL` calling-convention macro — defined in `q_shared.h`

# code/game/chars.h
## File Purpose
Defines integer constants (characteristic indices) used to index into a bot's personality/behavior data structure. Each constant maps a named behavioral trait to a slot number understood by the bot AI and botlib systems.

## Core Responsibilities
- Enumerate all bot characteristic slot indices (0–48)
- Categorize traits into logical groups: identity, combat, chat, movement, and goal-seeking
- Provide a shared vocabulary between the game module and botlib for reading/writing bot personality values

## External Dependencies
- No `#include` directives; this header is self-contained.
- **Defined elsewhere / consumers:**
  - `botlib/be_ai_char.c` — reads/writes characteristic values using these indices
  - `game/ai_main.c`, `game/ai_dmq3.c`, etc. — pass these constants to botlib API calls such as `trap_Characteristic_Float` / `trap_Characteristic_String`
  - `botlib/botlib.h` — declares the `BotCharacteristic_*` API that accepts these index values


# code/game/g_active.c
## File Purpose
Implements per-client per-frame logic for the server-side game module, covering player movement, environmental effects, damage feedback, event dispatch, and end-of-frame state synchronization. It is the central "think" driver for all connected clients each server frame.

## Core Responsibilities
- Run `Pmove` physics simulation for each client and propagate results back to entity state
- Apply world environmental damage (drowning, lava, slime) each frame
- Aggregate and encode damage feedback into `playerState_t` for pain blends/kicks
- Dispatch and process server-authoritative client events (falling, weapon fire, item use, teleport)
- Handle spectator movement and chase-cam follow logic
- Enforce inactivity kick timer and respawn conditions
- Execute once-per-second timer actions (health regen, armor decay, ammo regen via MISSIONPACK)
- Synchronize `playerState_t` → `entityState_t` and send predictable events to other clients

## External Dependencies
- `g_local.h` (pulls `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `Pmove` (bg_pmove.c), `G_Damage`, `G_AddEvent`, `G_Sound`, `G_TempEntity`, `G_SoundIndex` (g_utils/g_combat), `BG_PlayerStateToEntityState`, `BG_PlayerTouchesItem` (bg_misc.c), `FireWeapon`, `CheckGauntletAttack`, `Weapon_HookFree` (g_weapon.c), `TeleportPlayer`, `SelectSpawnPoint`, `respawn` (g_client/g_misc), `Drop_Item` (g_items.c), `BotTestAAS` (ai_main.c), all `trap_*` syscalls (g_syscalls.c)

# code/game/g_arenas.c
## File Purpose
Manages the post-game intermission sequence for Quake III Arena's single-player and tournament modes, including spawning player model replicas on victory podiums and assembling the `postgame` server command that drives the end-of-match scoreboard/stats UI.

## Core Responsibilities
- Collect and format end-of-match statistics into a `postgame` console command sent to all clients
- Spawn a physical podium entity in the intermission zone
- Spawn static player body replicas on the podium for the top 3 finishers
- Continuously reorient the podium and its occupants toward the intermission camera via a think function
- Drive the winner's celebration (gesture) animation with a timed start/stop
- Provide a server command (`Svcmd_AbortPodium_f`) to cancel the podium celebration in single-player

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `level` (`level_locals_t`), `g_entities[]`, `g_gametype`; trap functions (`trap_SendConsoleCommand`, `trap_LinkEntity`, `trap_Cvar_VariableIntegerValue`); math utilities (`AngleVectors`, `VectorMA`, `vectoangles`, `vectoyaw`); entity helpers (`G_Spawn`, `G_SetOrigin`, `G_ModelIndex`, `G_AddEvent`, `G_Printf`); `CalculateRanks`; `SP_PODIUM_MODEL` (defined in `g_local.h`).

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

## External Dependencies
- `g_local.h` — all shared game types, trap declarations, `level`, `g_entities`, cvars
- `BotAISetupClient`, `BotAIShutdown` — defined in `ai_main.c` (botlib AI layer)
- `ClientConnect`, `ClientBegin`, `ClientDisconnect` — defined in `g_client.c`
- `G_Alloc` — defined in `g_mem.c`
- `PickTeam` — defined in `g_client.c`
- `podium1/2/3` — `extern gentity_t*` owned by `g_arenas.c`
- `COM_Parse`, `COM_ParseExt`, `Info_SetValueForKey`, `Info_ValueForKey`, `Q_strncpyz` — defined in `q_shared.c` / `bg_lib.c`
- All `trap_*` functions — syscall stubs resolved by the VM/engine boundary

# code/game/g_client.c
## File Purpose
Manages the full client lifecycle within the game module: connection, spawning, respawning, userinfo updates, body queue management, and disconnection. Handles spawn point selection logic and player state initialization at each spawn.

## Core Responsibilities
- Spawn point registration (`SP_info_player_*`) and selection (nearest, random, furthest, initial, spectator)
- Body queue management: pooling corpse entities, animating their sink/disappearance
- Client lifecycle callbacks: `ClientConnect`, `ClientBegin`, `ClientSpawn`, `ClientDisconnect`
- Userinfo parsing and configstring broadcasting (`ClientUserinfoChanged`)
- Player name sanitization (`ClientCleanName`)
- Team utility queries: `TeamCount`, `TeamLeader`, `PickTeam`
- View angle delta computation (`SetClientViewAngle`)

## External Dependencies

- **Includes:** `g_local.h` (which pulls `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `ClientThink`, `ClientEndFrame` — `g_active.c`
  - `SelectCTFSpawnPoint` — `g_team.c`
  - `BG_PlayerStateToEntityState` — `bg_misc.c`
  - `FindIntermissionPoint`, `MoveClientToIntermission` — `g_main.c` / `p_hud.c`
  - `TossClientItems`, `body_die` — `g_combat.c`
  - `G_BotConnect`, `BotAIShutdownClient` — `g_bot.c` / `ai_main.c`
  - `CalculateRanks`, `BroadcastTeamChange` — `g_main.c` / `g_cmds.c`
  - All `trap_*` functions — server syscall stubs in `g_syscalls.c`

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

## External Dependencies
- **Includes:** `g_local.h` (all game types, trap functions, globals), `../../ui/menudef.h` (VOICECHAT_* string constants)
- **Defined elsewhere:** `level` (`level_locals_t` global from `g_main.c`), `g_entities` array, all `trap_*` syscalls (resolved by the engine VM), `player_die`, `BeginIntermission`, `TeleportPlayer`, `CopyToBodyQue`, `ClientUserinfoChanged`, `ClientBegin`, `SetLeader`, `CheckTeamLeader`, `PickTeam`, `TeamCount`, `TeamLeader`, `OnSameTeam`, `Team_GetLocationMsg`, `BG_FindItem`, `G_Spawn`, `G_SpawnItem`, `FinishSpawningItem`, `Touch_Item`, `G_FreeEntity`, `G_LogPrintf`, `G_Printf`, `G_Error`

# code/game/g_combat.c
## File Purpose
Implements all server-side combat logic for Quake III Arena's game module, including damage application, knockback, scoring, death processing, item drops, and radius explosion damage. It serves as the central damage pipeline that all weapons and hazards funnel through.

## Core Responsibilities
- Apply damage to entities via `G_Damage`, handling armor absorption, knockback, godmode, team protection, and invulnerability
- Execute player death sequence via `player_die`, including obituary logging, scoring, animation, and flag/item handling
- Perform area-of-effect damage via `G_RadiusDamage` with line-of-sight gating
- Drop held weapons and powerups on player death via `TossClientItems`
- Manage score additions and visual score plums via `AddScore`/`ScorePlum`
- Handle gib deaths and body corpse state transitions via `GibEntity`/`body_die`
- Detect near-capture/near-score events for "holy shit" reward triggers

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `g_entities[]`, `level` — global game state (`g_main.c`)
  - `g_knockback`, `g_blood`, `g_friendlyFire`, `g_gametype`, `g_debugDamage`, `g_cubeTimeout` — cvars
  - `Team_FragBonuses`, `Team_ReturnFlag`, `Team_CheckHurtCarrier`, `OnSameTeam` — `g_team.c`
  - `Drop_Item`, `LaunchItem`, `BG_FindItemForWeapon`, `BG_FindItemForPowerup`, `BG_FindItem` — items/bg layer
  - `Weapon_HookFree`, `LogAccuracyHit` — `g_weapon.c`
  - `Cmd_Score_f` — `g_cmds.c`
  - `G_StartKamikaze` — `g_weapon.c` (MISSIONPACK)
  - `CheckObeliskAttack` — `g_team.c` (MISSIONPACK)
  - All `trap_*` functions — syscall interface to the server engine

# code/game/g_items.c
## File Purpose
Implements the server-side item system for Quake III Arena, handling pickup logic, item spawning, dropping, respawning, and per-frame physics simulation for all in-game collectibles (weapons, ammo, health, armor, powerups, holdables, and team items).

## Core Responsibilities
- Execute type-specific pickup logic and award appropriate effects to the picking client
- Manage item respawn timers and team-based item selection on respawn
- Spawn world items at map load, dropping them to floor via trace
- Launch and drop items dynamically (e.g., on player death)
- Simulate per-frame physics for airborne items (gravity, bounce, NODROP removal)
- Maintain the item registration/precache bitfield written to config strings
- Validate required team-game entities (flags, obelisks) at map start

## External Dependencies
- `g_local.h` → `q_shared.h`, `bg_public.h`, `g_public.h`
- **Defined elsewhere:**
  - `bg_itemlist`, `bg_numItems` — item table (`bg_misc.c`)
  - `BG_CanItemBeGrabbed`, `BG_FindItem`, `BG_FindItemForWeapon`, `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — shared game library
  - `Pickup_Team`, `Team_DroppedFlagThink`, `Team_CheckDroppedItem`, `Team_FreeEntity`, `Team_InitGame` — `g_team.c`
  - `G_Spawn`, `G_FreeEntity`, `G_TempEntity`, `G_UseTargets`, `G_SetOrigin`, `G_AddEvent`, `G_AddPredictableEvent`, `G_SoundIndex`, `G_RunThink` — `g_utils.c` / `g_main.c`
  - `trap_Trace`, `trap_LinkEntity`, `trap_PointContents`, `trap_SetConfigstring`, `trap_Cvar_VariableIntegerValue`, `trap_GetUserinfo` — engine syscall stubs
  - `g_weaponRespawn`, `g_weaponTeamRespawn`, `g_gametype` — cvars declared in `g_main.c`
  - `level` — global `level_locals_t` from `g_main.c`

# code/game/g_local.h
## File Purpose
Central private header for the Quake III Arena server-side game module (game DLL/VM). It defines all major game-side data structures, declares every cross-file function, enumerates game cvars, and lists the full `trap_*` syscall interface that bridges the game VM to the engine.

## Core Responsibilities
- Define `gentity_t` (the universal server-side entity) and `gclient_t` (per-client runtime state)
- Define `level_locals_t`, the singleton that holds all per-map game state
- Define `clientPersistant_t` and `clientSession_t` for data surviving respawns/levels
- Declare every public function exported between game `.c` files
- Declare all `vmCvar_t` globals used by the game module
- Declare all `trap_*` engine syscall wrappers (filesystem, collision, bot AI, etc.)
- Define entity flag bits (`FL_*`), damage flags (`DAMAGE_*`), and timing constants

## External Dependencies

- **Includes**: `q_shared.h` (base types, math, `entityState_t`, `playerState_t`), `bg_public.h` (shared game types: items, weapons, pmove, events), `g_public.h` (engine API enum, `sharedEntity_t`, `entityShared_t`), `g_team.h` (CTF/team function prototypes)
- **Defined elsewhere**:
  - `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t` — `q_shared.h`
  - `entityShared_t`, `gameImport_t` — `g_public.h`
  - `gitem_t`, `weapon_t`, `team_t`, `gametype_t` — `bg_public.h`
  - All `trap_*` function bodies — `g_syscalls.c` (VM syscall dispatch stubs)
  - `level`, `g_entities`, all `vmCvar_t` definitions — `g_main.c`

# code/game/g_main.c
## File Purpose
The central game module entry point for Quake III Arena's server-side game logic. It owns the VM dispatch table (`vmMain`), manages game initialization/shutdown, drives the per-frame update loop, and maintains all game-wide cvars and level state.

## Core Responsibilities
- Expose `vmMain` as the sole entry point from the engine into the game VM
- Register and update all server-side cvars via `gameCvarTable`
- Initialize and tear down the game world (`G_InitGame`, `G_ShutdownGame`)
- Drive the per-frame entity update loop (`G_RunFrame`)
- Manage tournament warmup, voting, team voting, and exit rules
- Compute and broadcast player/team score rankings
- Handle level intermission sequencing and map transitions

## External Dependencies
- `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`) — all shared types and trap declarations
- `trap_*` syscalls — defined in the engine, bridged through `g_syscalls.c`; cover FS, cvars, server commands, entity linking, AAS, bot lib, etc.
- `ClientConnect`, `ClientThink`, `ClientBegin`, `ClientDisconnect`, `ClientCommand`, `ClientUserinfoChanged`, `ClientEndFrame` — defined in `g_client.c` / `g_active.c`
- `BotAISetup`, `BotAIShutdown`, `BotAILoadMap`, `BotAIStartFrame`, `BotInterbreedEndMatch` — defined in `ai_main.c` / `g_bot.c`
- `G_SpawnEntitiesFromString`, `G_CheckTeamItems`, `UpdateTournamentInfo`, `SpawnModelsOnVictoryPads`, `CheckTeamStatus` — defined elsewhere in the game module

# code/game/g_mem.c
## File Purpose
Provides a simple bump-pointer memory allocator backed by a fixed 256 KB static pool for the game module. All allocations are permanent for the duration of a map session; there is no free operation.

## Core Responsibilities
- Allocate memory from a fixed-size static pool with 32-byte alignment
- Detect pool exhaustion and fatal-error on overflow
- Reset the pool at map/session start via `G_InitMemory`
- Expose current pool usage via a server console command

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `g_debugAlloc` — `vmCvar_t` extern declared in `g_local.h`, defined in `g_main.c`
  - `G_Printf` — defined in `g_main.c`; wraps `trap_Printf`
  - `G_Error` — defined in `g_main.c`; wraps `trap_Error` (non-returning)

# code/game/g_misc.c
## File Purpose
Implements miscellaneous map entity spawn functions and gameplay systems for the Quake III Arena game module, including teleportation logic, portal surfaces, positional markers, and trigger-based weapon shooters.

## Core Responsibilities
- Spawn and initialize editor-only or utility entities (`info_null`, `info_camp`, `light`, `func_group`)
- Implement the `TeleportPlayer` function used by trigger teleporters and portals
- Set up portal surface/camera pairs for in-world mirror/portal rendering
- Initialize trigger-based weapon shooter entities (`shooter_rocket`, `shooter_plasma`, `shooter_grenade`)
- Handle `#ifdef MISSIONPACK` portal item mechanics (drop source/destination pads)

## External Dependencies
- **`g_local.h`** — pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, all `gentity_t`/`gclient_t` definitions, and all `trap_*` syscall declarations.
- **Defined elsewhere:** `G_TempEntity`, `G_KillBox`, `G_PickTarget`, `G_SetMovedir`, `BG_PlayerStateToEntityState`, `SetClientViewAngle`, `fire_grenade`, `fire_rocket`, `fire_plasma`, `RegisterItem`, `BG_FindItemForWeapon`, `Drop_Item`, `BG_FindItemForPowerup`, `G_Damage`, `G_Find`, `G_Spawn`, `G_SetOrigin`, `G_FreeEntity`, `DirToByte`, `PerpendicularVector`, `CrossProduct`, `crandom`, `level` (global), all `trap_*` functions.

# code/game/g_missile.c
## File Purpose
Implements server-side missile entity creation, movement simulation, and impact handling for all projectile weapons in Quake III Arena. It spawns missile entities, advances them each frame via trajectory evaluation and collision tracing, and dispatches bounce, impact, or explosion logic on collision.

## Core Responsibilities
- Spawn typed missile entities (plasma, grenade, rocket, BFG, grapple, and MISSIONPACK: nail, prox mine)
- Advance missiles each server frame: evaluate trajectory, trace movement, detect collisions
- Handle missile impact: apply direct damage, splash damage, bounce, grapple attachment
- Manage MISSIONPACK proximity mine lifecycle: activation, trigger volumes, player-sticking, timed explosion
- Emit network events (hit/miss/bounce/explosion) for client-side effects
- Track accuracy hits on the owning client

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — `bg_pmove.c`/`bg_misc.c`
  - `G_Damage`, `G_RadiusDamage`, `CanDamage`, `G_InvulnerabilityEffect` — `g_combat.c`
  - `LogAccuracyHit`, `Weapon_HookFree`, `Weapon_HookThink`, `SnapVectorTowards` — `g_weapon.c`
  - `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `G_AddEvent`, `G_SoundIndex` — `g_utils.c`
  - `G_RunThink` — `g_main.c`
  - `trap_Trace`, `trap_LinkEntity` — engine syscall stubs (`g_syscalls.c`)
  - `level`, `g_entities`, `g_proxMineTimeout`, `g_gametype` — game module globals

# code/game/g_mover.c
## File Purpose
Implements all moving entity (mover) logic for Quake III Arena's game module, including the push/collision system for movers and spawn functions for doors, platforms, buttons, trains, and decorative movers (rotating, bobbing, pendulum, static).

## Core Responsibilities
- Execute per-frame movement for mover entities via `G_RunMover` / `G_MoverTeam`
- Push (or block) entities that intersect a moving brush, with full rollback on failure
- Manage binary mover state transitions (POS1 ↔ POS2) and associated sounds/events
- Spawn and configure all `func_*` mover entity types from map data
- Handle door trigger volumes, spectator teleportation through doors, and platform touch logic
- Synchronize team-linked mover slaves so all parts move atomically

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `g_entities[]`, `level` (level_locals_t), `BG_EvaluateTrajectory`, `RadiusFromBounds`, `AngleVectors`, `VectorInverse`, `trap_*` syscalls, `G_Damage`, `G_AddEvent`, `G_UseTargets`, `G_Find`, `G_Spawn`, `G_FreeEntity`, `TeleportPlayer`, `Team_DroppedFlagThink`, `G_ExplodeMissile`, `G_RunThink`, `g_gravity` (vmCvar_t)

# code/game/g_public.h
## File Purpose
Defines the public interface contract between the Quake III game module (QVM) and the server engine. It declares server-visible entity flags, shared entity data structures, and the complete syscall tables for both engine-to-game (imports) and game-to-engine (exports) communication.

## Core Responsibilities
- Define `GAME_API_VERSION` for versioning the game/server ABI
- Declare `SVF_*` bitflags controlling server-side entity visibility and behavior
- Define `entityShared_t` and `sharedEntity_t` as the shared memory layout the server reads directly
- Enumerate all engine syscalls available to the game module (`gameImport_t`)
- Enumerate all entry points the server calls into the game module (`gameExport_t`)
- Expose BotLib syscall ranges (200–599) as part of the game import table

## External Dependencies
- `entityState_t`, `playerState_t`, `usercmd_t`, `trace_t`, `vec3_t`, `vmCvar_t`, `qboolean` — defined in `q_shared.h` / `bg_public.h` (game-shared layer)
- `gentity_t` — defined in `g_local.h`; `g_public.h` only sees it as a forward-referenced pointer target through `sharedEntity_t`
- Server engine — consumes this header to understand entity layout and dispatch the VM syscall tables
- BotLib — its full API surface is tunneled through the `gameImport_t` enum rather than direct linking

# code/game/g_rankings.c
## File Purpose
Implements the game-side interface to Quake III Arena's global online rankings system, collecting and submitting per-player statistics (weapon usage, damage, deaths, pickups, rewards) to an external ranking service via trap calls during and at the end of each match.

## Core Responsibilities
- Drive the rankings subsystem each server frame (init, poll, status management)
- Enforce ranked-game rules (kick bots, cap timelimit/fraglimit)
- Track and submit per-player combat statistics: shots fired, hits given/taken, damage, splash
- Report death events classified as frags, suicides, or hazard kills
- Report item pickups (weapons, ammo, health, armor, powerups, holdables)
- Report time spent with each weapon equipped
- Finalize and submit match-level metadata on game-over

## External Dependencies
- **Includes:** `g_local.h` (game entity/client types, level globals, all trap declarations), `g_rankings.h` (QGR_KEY_* constants, `GR_GAMEKEY`)
- **Defined elsewhere:** `trap_RankCheckInit`, `trap_RankBegin`, `trap_RankPoll`, `trap_RankActive`, `trap_RankUserStatus`, `trap_RankUserReset`, `trap_RankReportInt`, `trap_RankReportStr` — ranking system trap calls into the engine/VM syscall layer; `level` (`level_locals_t`), `g_entities[]` — game globals; `ClientSpawn`, `SetTeam`, `DeathmatchScoreboardMessage`, `OnSameTeam` — other game module functions; `GR_GAMEKEY` — game-key constant (defined elsewhere, not in the provided headers)

# code/game/g_rankings.h
## File Purpose
Defines a comprehensive set of numeric key constants used to report per-player and per-session statistics to a global online rankings/scoring backend. Each key encodes metadata about the stat's type, aggregation method, and category directly within its numeric value.

## Core Responsibilities
- Define all `QGR_KEY_*` constants for the rankings reporting system
- Encode stat semantics (report type, stat type, data type, calculation method, category) into each key's decimal digits
- Provide per-weapon stat keys for all 10 base weapons (Gauntlet through Grapple) plus unknowns
- Conditionally define `MISSIONPACK`-exclusive keys for Team Arena weapons, ammo, powerups, and holdables
- Provide keys for session metadata (hostname, map, gametype, limits)
- Provide keys for hazards, rewards, CTF events, and teammate interaction

## External Dependencies

- No includes.
- The key encoding scheme implies an external global rankings server/API (not defined here) that interprets the numeric key structure.
- `MISSIONPACK` macro defined externally (build system / project settings) to enable Team Arena extensions.

---

**Key encoding schema** (decoded from the header comment):

| Digit position | Meaning | Notable values |
|---|---|---|
| 10⁹ | Report type | 1=normal, 2=dev-only |
| 10⁸ | Stat type | 0=match, 1=single-player, 2=duel |
| 10⁷ | Data type | 0=string, 1=uint32 |
| 10⁶ | Calculation | 0=raw, 1=add, 2=avg, 3=max, 4=min |
| 10⁴–10⁵ | Category | 00=general, 02=weapon, 09=reward, 11=CTF, etc. |
| 10²–10³ | Sub-category | weapon index (×100) or item tier |
| 10⁰–10¹ | Ordinal | stat variant within category |

# code/game/g_session.c
## File Purpose
Manages persistent client session data in Quake III Arena's server-side game module. Session data survives across level loads and tournament restarts by serializing to and deserializing from cvars at shutdown/reconnect time.

## Core Responsibilities
- Serialize per-client session state to named cvars on game shutdown
- Deserialize per-client session state from cvars on reconnect
- Initialize fresh session data for first-time connecting clients
- Initialize the world session and detect gametype changes across sessions
- Write all connected clients' session data atomically at shutdown

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer` — engine syscall stubs (g_syscalls.c)
  - `PickTeam`, `BroadcastTeamChange` — defined in `g_client.c` / `g_cmds.c`
  - `Info_ValueForKey` — defined in `q_shared.c`
  - `va` — defined in `q_shared.c`
  - `level`, `g_gametype`, `g_teamAutoJoin`, `g_maxGameClients` — globals defined in `g_main.c`

# code/game/g_spawn.c
## File Purpose
Parses the map's entity string at level load time, translates key/value spawn variables into binary `gentity_t` fields, and dispatches each entity to its class-specific spawn function. It is the entry point for all server-side entity instantiation from BSP data.

## Core Responsibilities
- Read and store raw key/value token pairs from the BSP entity string (`G_ParseSpawnVars`)
- Provide typed accessors for spawn variables: string, float, int, vector (`G_SpawnString`, etc.)
- Map string field names to `gentity_t` struct offsets and write typed values (`G_ParseField`)
- Look up and invoke the correct spawn function by classname (`G_CallSpawn`)
- Process the `worldspawn` entity to apply global level settings (`SP_worldspawn`)
- Filter entities by gametype flags (`notsingle`, `notteam`, `notfree`, `notq3a`/`notta`, `gametype`)
- Drive the full entity spawning loop for an entire level (`G_SpawnEntitiesFromString`)

## External Dependencies
- `g_local.h` — `gentity_t`, `level_locals_t`, `FOFS`, all `g_*` cvars, all `trap_*` syscalls
- `bg_public.h` (via `g_local.h`) — `bg_itemlist`, `gitem_t`, gametype constants (`GT_*`)
- **Defined elsewhere:** `G_Spawn`, `G_FreeEntity`, `G_Alloc`, `G_SpawnItem`, `G_Error`, `G_Printf`, `G_LogPrintf`, `trap_GetEntityToken`, `trap_SetConfigstring`, `trap_Cvar_Set`, `Q_stricmp`, all `SP_*` spawn functions (defined in `g_misc.c`, `g_mover.c`, `g_trigger.c`, `g_target.c`, `g_items.c`, etc.)

# code/game/g_svcmds.c
## File Purpose
Implements server-console-only commands for the Quake III Arena game module, including IP-based packet filtering/banning and administrative commands such as entity listing, team forcing, and bot management dispatch.

## Core Responsibilities
- Maintain an in-memory IP filter list (`ipFilters[]`) for allow/deny packet filtering
- Parse and persist IP ban masks to/from the `g_banIPs` cvar string
- Provide `G_FilterPacket` to gate incoming connections against the filter list
- Expose `Svcmd_AddIP_f` / `Svcmd_RemoveIP_f` for runtime ban management
- Implement `ConsoleCommand` as the single dispatch entry point for all server-console commands
- Provide `ClientForString` helper to resolve a client by slot number or name

## External Dependencies
- **Includes:** `g_local.h` (which transitively brings in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `trap_Argv`, `trap_Argc`, `trap_Cvar_Set`, `trap_SendConsoleCommand`, `trap_SendServerCommand` — VM syscall stubs (`g_syscalls.c`)
  - `G_Printf`, `Com_Printf` — logging (`g_main.c` / engine)
  - `SetTeam` — `g_cmds.c`
  - `ConcatArgs` — `g_cmds.c` (declared but not defined here)
  - `Svcmd_GameMem_f` — `g_mem.c`; `Svcmd_AddBot_f`, `Svcmd_BotList_f` — `g_bot.c`; `Svcmd_AbortPodium_f` — `g_arenas.c`
  - `g_filterBan`, `g_banIPs`, `g_dedicated` — cvars declared in `g_local.h`, registered in `g_main.c`
  - `level`, `g_entities` — global game state (`g_main.c`)

# code/game/g_syscalls.c
## File Purpose
Implements the DLL-side system call interface for the game module, providing typed C wrapper functions around a single variadic `syscall` function pointer set by the engine at load time. This file is excluded from QVM builds, where `g_syscalls.asm` is used instead.

## Core Responsibilities
- Receive and store the engine's syscall dispatch function pointer via `dllEntry`
- Wrap every engine API call (file I/O, cvars, networking, collision, etc.) as typed C functions
- Bridge float arguments through `PASSFLOAT` to avoid ABI issues with variadic integer-only syscall conventions
- Expose the full BotLib/AAS API surface to game logic via trap functions
- Provide entity action (EA) wrappers for bot input simulation

## External Dependencies
- `code/game/g_local.h` — pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, and all game type/enum definitions
- `G_PRINT`, `G_ERROR`, `G_LOCATE_GAME_DATA`, `BOTLIB_*`, `G_TRACE`, etc. — syscall opcode enumerations defined in `g_public.h` / `botlib.h` (defined elsewhere)
- `gentity_t`, `playerState_t`, `trace_t`, `vmCvar_t`, `usercmd_t`, `pc_token_t` — defined elsewhere
- `QDECL` — calling convention macro, defined in `q_shared.h`

# code/game/g_target.c
## File Purpose
Implements all `target_*` entity types for Quake III Arena's server-side game logic. These are invisible map entities that perform actions (give items, print messages, play sounds, fire lasers, teleport players, etc.) when triggered by other entities or players.

## Core Responsibilities
- Register spawn functions (`SP_target_*`) for each target entity class
- Assign `use` callbacks that execute when the entity is triggered
- Implement delayed firing, score modification, and message broadcasting
- Manage looping/one-shot audio via `target_speaker`
- Operate a continuous damage laser (`target_laser`) with per-frame think logic
- Teleport activating players to a named destination entity
- Link `target_location` entities into a global linked list for HUD location display

## External Dependencies
- **`g_local.h`** — `gentity_t`, `level_locals_t`, `gclient_t`, all trap/utility declarations
- **Defined elsewhere:** `Touch_Item` (g_items.c), `Team_ReturnFlag` (g_team.c), `G_UseTargets`, `G_Find`, `G_PickTarget`, `G_SetMovedir`, `G_AddEvent`, `G_SoundIndex`, `G_SetOrigin` (g_utils.c), `TeleportPlayer` (g_misc.c), `G_Damage` (g_combat.c), `AddScore` (g_client.c), `G_TeamCommand` (g_utils.c), all `trap_*` syscalls

# code/game/g_team.c
## File Purpose
Implements all server-side team game logic for Quake III Arena, covering CTF flag lifecycle (pickup, drop, capture, return), team scoring, frag bonuses, player location tracking, spawn point selection, and MISSIONPACK obelisk/harvester mechanics.

## Core Responsibilities
- Manage CTF and One-Flag-CTF flag state (at base, dropped, taken, captured)
- Award frag bonuses for flag carrier kills, carrier defense, and base defense
- Broadcast team sound events on score changes, flag events, and obelisk attacks
- Track and broadcast team overlay info (health, armor, weapon, location) per frame
- Provide team spawn point selection for CTF game starts and respawns
- Handle obelisk entity lifecycle: spawning, regen, pain, death, respawn (MISSIONPACK)
- Register map spawn entities for CTF player/spawn spots and obelisks

## External Dependencies
- `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, `g_team.h`)
- **Defined elsewhere:** `AddScore`, `CalculateRanks`, `G_Find`, `G_TempEntity`, `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `RespawnItem`, `SelectSpawnPoint`, `SpotWouldTelefrag`, `trap_SetConfigstring`, `trap_SendServerCommand`, `trap_InPVS`, `trap_Trace`, `trap_LinkEntity`, `level` (global), `g_entities` (global), all `g_obelisk*` cvars.

# code/game/g_team.h
## File Purpose
Header file for the Quake III Arena team-based game mode (CTF and Missionpack variants). It defines scoring constants for Capture the Flag mechanics and declares the public interface for team logic used by the server-side game module.

## Core Responsibilities
- Declares CTF scoring bonus constants, conditionally compiled for MISSIONPACK vs. base Q3A balancing
- Declares geometric radius and timing constants for proximity-based bonus logic
- Declares grapple hook physics constants
- Exposes the public function interface for all team/CTF game logic to the rest of the game module

## External Dependencies
- `gentity_t`, `team_t`, `vec3_t`, `qboolean` — defined in `g_local.h` / `q_shared.h`
- `MISSIONPACK` — preprocessor define controlling two distinct scoring balance sets; defined at build time
- All function bodies defined in `g_team.c`

# code/game/g_trigger.c
## File Purpose
Implements all map trigger entities for Quake III Arena's server-side game module. Handles volume-based activation, jump pads, teleporters, hurt zones, and repeating timers that fire targets when players or entities interact with them.

## Core Responsibilities
- Initialize trigger brush entities with correct collision contents and server flags
- Implement `trigger_multiple`: repeatable volume trigger with optional team filtering and wait/random timing
- Implement `trigger_always`: fires targets once on map load, then frees itself
- Implement `trigger_push` / `target_push`: jump pad physics, computing launch velocity to hit a target apex
- Implement `trigger_teleport`: client-predicted teleport volumes, with optional spectator-only mode
- Implement `trigger_hurt`: damage zones with SLOW/SILENT/NO_PROTECTION/START_OFF flags
- Implement `func_timer`: a non-spatial, toggleable repeating timer that fires targets

## External Dependencies
- **`g_local.h`**: `gentity_t`, `gclient_t`, `level_locals_t` (`level`), `g_gravity`, `FRAMETIME`, `CONTENTS_TRIGGER`, `SVF_NOCLIENT`, `TEAM_RED/BLUE/SPECTATOR`, `ET_PUSH_TRIGGER`, `ET_TELEPORT_TRIGGER`, damage flags, `MOD_TRIGGER_HURT`
- **Defined elsewhere:** `G_UseTargets`, `G_PickTarget`, `G_FreeEntity`, `G_SetMovedir`, `G_Sound`, `G_SoundIndex`, `G_Damage`, `TeleportPlayer`, `BG_TouchJumpPad`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_SetBrushModel`, `crandom`, `G_SpawnFloat`, `G_Printf`

# code/game/g_utils.c
## File Purpose
Provides core utility functions for the Quake III Arena server-side game module, including entity lifecycle management (spawn, free, temp entities), entity search/targeting, event signaling, shader remapping, and miscellaneous math/string helpers.

## Core Responsibilities
- Entity allocation (`G_Spawn`), initialization (`G_InitGentity`), and deallocation (`G_FreeEntity`)
- Temporary event-entity creation (`G_TempEntity`)
- Entity search by field offset (`G_Find`) and random target selection (`G_PickTarget`)
- Target chain activation (`G_UseTargets`) and team-broadcast commands (`G_TeamCommand`)
- Game event attachment to entities (`G_AddEvent`, `G_AddPredictableEvent`)
- Shader remapping table management (`AddRemap`, `BuildShaderStateConfig`)
- Configstring index registration for models and sounds (`G_FindConfigstringIndex`)

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `g_entities[]`, `level` (`level_locals_t`), all `trap_*` syscall stubs, `G_Damage`, `BG_AddPredictableEventToPlayerstate`, `AngleVectors`, `VectorCompare`, `Com_sprintf`, `Q_stricmp`, `Q_strcat`, `SnapVector`

# code/game/g_weapon.c
## File Purpose
Implements all server-side weapon firing logic for Quake III Arena, translating player weapon inputs into world-space traces, damage events, and projectile spawns. It is the authoritative damage source for hitscan weapons and the launch point for projectile entities.

## Core Responsibilities
- Compute muzzle position and firing direction from player view state
- Execute hitscan traces for gauntlet, machinegun, shotgun, railgun, and lightning gun
- Spawn projectile entities for rocket, grenade, plasma, BFG, grapple (and MissionPack: nail, prox mine)
- Apply Quad Damage (and MISSIONPACK Doubler) multipliers to all outgoing damage
- Track per-client shot/hit accuracy counters; award "Impressive" for back-to-back railgun hits
- Emit temp entities (EV_BULLET_HIT_FLESH, EV_RAILTRAIL, EV_SHOTGUN, etc.) for client-side effects
- MISSIONPACK: handle Kamikaze holdable item with expanding radius damage and shockwave

## External Dependencies
- `g_local.h` → `q_shared.h`, `bg_public.h`, `g_public.h` (all game types and trap declarations)
- **Defined elsewhere:** `g_entities[]`, `level` (globals in `g_main.c`); `fire_rocket`, `fire_grenade`, `fire_plasma`, `fire_bfg`, `fire_grapple`, `fire_nail`, `fire_prox` (`g_missile.c`); `G_Damage`, `G_InvulnerabilityEffect` (`g_combat.c`); `OnSameTeam` (`g_team.c`); `g_quadfactor`, `g_gametype` (cvars registered in `g_main.c`); `trap_Trace`, `trap_LinkEntity`, `trap_UnlinkEntity`, `trap_EntitiesInBox` (engine syscalls)

# code/game/inv.h
## File Purpose
A pure C header defining integer constants for inventory slots, item model indices, and weapon indices used by the bot AI system. It serves as a shared lookup table mapping game items to numeric identifiers consumed by botlib's fuzzy logic and goal-evaluation scripts.

## Core Responsibilities
- Defines `INVENTORY_*` slot indices for armor, weapons, ammo, powerups, and flags/cubes used by bot AI inventory queries
- Defines enemy awareness constants (`ENEMY_HORIZONTAL_DIST`, `ENEMY_HEIGHT`, `NUM_VISIBLE_*`) as pseudo-inventory fuzzy inputs
- Defines `MODELINDEX_*` constants that must stay synchronized with the `bg_itemlist` array in `bg_misc.c`
- Defines `WEAPONINDEX_*` constants mapping logical weapon slots to 1-based integer IDs

## External Dependencies
- **`bg_misc.c`** — `bg_itemlist[]` array ordering must exactly match the `MODELINDEX_*` sequence; a mismatch silently corrupts bot item recognition
- **`MISSIONPACK`** — conditional compilation guard present but body is empty (`#error` is commented out); mission pack items (`INVENTORY_KAMIKAZE`, `MODELINDEX_KAMIKAZE`, etc.) are defined unconditionally regardless of the guard


# code/game/match.h
## File Purpose
This header defines all symbolic constants used by the bot AI's natural-language chat matching and team-command messaging system. It provides message type identifiers, match-template context flags, command sub-type bitmasks, and variable-slot indices that map parsed chat tokens to structured bot commands.

## Core Responsibilities
- Define the escape character (`EC`) used to delimit in-game chat tokens
- Declare bitmask flags for match-template parsing contexts (e.g., CTF, teammate address, time)
- Enumerate all bot-to-bot and bot-to-player message type codes (`MSG_*`)
- Provide command sub-type bitmask flags (`ST_*`) for qualifying message semantics
- Define named indices for word-replacement variable slots in message templates

## External Dependencies
- No includes in this file itself.
- Consumed by: `code/game/ai_chat.c`, `code/game/ai_cmd.c`, `code/game/ai_team.c`, and related bot source files (defined elsewhere).
- `EC` (`"\x19"`) must match the escape character literal used in chat string definitions in `g_cmd.c` (comment-enforced contract, not compiler-enforced).

---

**Notes:**
- `ST_1FCTFGOTFLAG` (`65535` / `0xFFFF`) appears to be a sentinel or "all flags set" value rather than a single-bit flag — its use among power-of-two `ST_*` values suggests a special aggregate case for one-flag CTF mode.
- Several `#define` names collide in value (e.g., `THE_ENEMY` and `THE_TEAM` are both `7`; `FLAG` and `PLACE` are both `1`; `ADDRESSEE` and `MESSAGE` are both `2`) — these are intentional aliasing of variable-slot indices for different message contexts, not bugs.
- `MSG_WHOISTEAMLAEDER` contains a typo ("LAEDER" instead of "LEADER") preserved from the original id Software source.

# code/game/q_math.c
## File Purpose
Stateless mathematical utility library shared across all Quake III Arena modules (game, cgame, UI, renderer). Provides 3D vector math, angle conversion, plane operations, bounding box utilities, and fast approximation routines.

## Core Responsibilities
- Vector arithmetic: normalize, dot/cross product, rotate, scale, MA operations
- Angle utilities: conversion, normalization, interpolation, delta computation
- Plane operations: construction from points, sign-bit classification, box-plane side testing
- Bounding box management: clear, expand, radius computation
- Direction compression: float normal ↔ quantized byte index via `bytedirs` table
- Fast math approximations: `Q_rsqrt` (Quake fast inverse square root), `Q_fabs`
- Seeded PRNG: `Q_rand`, `Q_random`, `Q_crandom`

## External Dependencies
- **Includes**: `q_shared.h` (all type definitions, macros, inline variants)
- **Defined elsewhere**: `assert`, `sqrt`, `cos`, `sin`, `atan2`, `fabs`, `isnan` from `<math.h>`; `memcpy`, `memset` from `<string.h>`; `VectorNormalize` (called by `PerpendicularVector`, defined later in same file); `PerpendicularVector` (called by `RotatePointAroundVector`, defined later in same file — forward reference resolved at link time within TU)
- **Platform asm paths**: x86 MSVC `__declspec(naked)` `BoxOnPlaneSide`; Linux/FreeBSD i386 uses external asm (excluded via `#if` guard)

# code/game/q_shared.c
## File Purpose
A stateless utility library compiled into every Quake III code module (game, cgame, ui, botlib). It provides portable string handling, text parsing, byte-order swapping, formatted output, and info-string manipulation that must be available in all execution environments including the QVM.

## Core Responsibilities
- Clamping, path, and file extension utilities
- Byte-order swap primitives for cross-platform endianness handling
- Tokenizing text parser with comment stripping and line tracking
- Safe string library replacements (`Q_str*`, `Q_strncpyz`, etc.)
- Color-sequence-aware string utilities (`Q_PrintStrlen`, `Q_CleanStr`)
- `va()` / `Com_sprintf()` formatted print helpers
- Info-string key/value encoding, lookup, insertion, and removal

## External Dependencies
- `#include "q_shared.h"` — all type definitions, macros, and prototypes.
- `Com_Error`, `Com_Printf` — defined in `qcommon/common.c` (host side) or provided via syscall trap in VM modules.
- Standard C: `vsprintf`, `strncpy`, `strlen`, `strchr`, `strcmp`, `strcpy`, `strcat`, `atof`, `tolower`, `toupper`.

# code/game/q_shared.h
## File Purpose
The universal shared header included first by all Quake III Arena program modules (game, cgame, UI, botlib, renderer, and tools). It defines the engine's foundational type system, math library, string utilities, network-communicated data structures, and cross-platform portability layer. Mod authors must never modify this file.

## Core Responsibilities
- Cross-platform portability: compiler warnings, CPU detection, `QDECL`, `ID_INLINE`, `PATH_SEP`, byte-order swap functions
- Primitive type aliases (`byte`, `qboolean`, `qhandle_t`, `vec_t`, `vec3_t`, etc.)
- Math library: vector/angle/matrix macros and inline functions, `Q_rsqrt`, `Q_fabs`, bounding-box helpers
- String utilities: `Q_stricmp`, `Q_strncpyz`, color-sequence stripping, `va()`, `Com_sprintf`
- Engine data structures communicated over the network: `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `gameState_t`
- Cvar system interface: `cvar_t`, `vmCvar_t`, and all `CVAR_*` flag bits
- Collision primitives: `cplane_t`, `trace_t`, `markFragment_t`
- Info-string key/value API declarations
- VM compatibility: conditionally includes `bg_lib.h` instead of standard C headers when compiled for the Q3 virtual machine

## External Dependencies
- `bg_lib.h` — VM-only C standard library replacement (included conditionally)
- `surfaceflags.h` — `CONTENTS_*` and `SURF_*` bitmask constants shared with q3map
- Standard C headers (`assert.h`, `math.h`, `stdio.h`, `stdarg.h`, `string.h`, `stdlib.h`, `time.h`, `ctype.h`, `limits.h`) — native builds only
- **Defined elsewhere:** `ShortSwap`, `LongSwap`, `FloatSwap` (byte-order helpers in `q_shared.c`); `Q_rsqrt`, `Q_fabs` on x86 (`q_math.c`); all `extern vec3_t`/`vec4_t` globals (`q_shared.c`); `Hunk_Alloc`/`Hunk_AllocDebug` (engine hunk allocator); `Com_Error`, `Com_Printf` (implemented per-module in engine/game/cgame/ui)

# code/game/surfaceflags.h
## File Purpose
Defines bitmask constants for brush content types and surface properties shared across the game engine, tools (BSP compiler, bot library), and utilities. The comment explicitly states it must be kept identical in both the quake and utils directories.

## Core Responsibilities
- Define `CONTENTS_*` flags describing what a brush volume contains (solid, liquid, clip, portal, etc.)
- Define `SURF_*` flags describing per-surface rendering and gameplay properties
- Serve as a shared contract between the game module, renderer, collision system, bot library, and map compiler tools

## External Dependencies
- Mirrored (must stay in sync) in `code/game/q_shared.h` — the comment warns these definitions also need to be there.
- Referenced by: `code/qcommon/cm_load.c`, `code/game/bg_pmove.c`, `code/renderer/tr_*.c`, `code/botlib/be_aas_*.c`, `q3map/` compiler sources, `code/bspc/` sources.
- No includes — this file is a pure constant-definition leaf with no dependencies of its own.


# code/game/syn.h
## File Purpose
Defines bitmask constants for bot chat context flags used by the AI chat system. These flags identify the situational context in which a bot chat synonym or response is valid.

## Core Responsibilities
- Define a bitmask enumeration of chat/behavior contexts for the bot AI
- Distinguish team-specific contexts (CTF red/blue, Obelisk, Harvester)
- Provide a catch-all `CONTEXT_ALL` mask for context-agnostic entries

## External Dependencies
- No includes or external symbols. Standalone macro-only header.

---

**Notes on constants:**

| Constant | Value | Meaning |
|---|---|---|
| `CONTEXT_ALL` | `0xFFFFFFFF` | Matches any context |
| `CONTEXT_NORMAL` | `1` | Default/generic context |
| `CONTEXT_NEARBYITEM` | `2` | Bot is near an item |
| `CONTEXT_CTFREDTEAM` | `4` | CTF, red team |
| `CONTEXT_CTFBLUETEAM` | `8` | CTF, blue team |
| `CONTEXT_REPLY` | `16` | Replying to another chat message |
| `CONTEXT_OBELISKREDTEAM` | `32` | Overload gametype, red team |
| `CONTEXT_OBELISKBLUETEAM` | `64` | Overload gametype, blue team |
| `CONTEXT_HARVESTERREDTEAM` | `128` | Harvester gametype, red team |
| `CONTEXT_HARVESTERBLUETEAM` | `256` | Harvester gametype, blue team |
| `CONTEXT_NAMES` | `1024` | Context for name-specific synonyms |

Values are powers of two, designed to be OR-combined into a composite context mask for lookup and filtering.

