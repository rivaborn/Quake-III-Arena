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

## Key Types / Data Structures
None defined here; all types are defined elsewhere and used by reference.

| Name | Kind | Purpose |
|---|---|---|
| `bot_state_t` | struct (defined elsewhere) | Per-bot state used by nearly every function |
| `bot_waypoint_t` | struct (defined elsewhere) | Linked list node for named spatial waypoints |
| `bot_goal_t` | struct (defined elsewhere) | AAS goal descriptor (area, origin, flags) |
| `bot_moveresult_t` | struct (defined elsewhere) | Result of a movement attempt |
| `bot_activategoal_t` | struct (defined elsewhere) | Goal for activating/unblocking entities |
| `aas_entityinfo_t` | struct (defined elsewhere) | AAS-level entity info used in entity predicate queries |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `gametype` | `int` | global (extern) | Current game type (FFA, CTF, etc.) |
| `maxclients` | `int` | global (extern) | Maximum number of connected clients |
| `bot_grapple` | `vmCvar_t` | global (extern) | Controls whether bots use the grappling hook |
| `bot_rocketjump` | `vmCvar_t` | global (extern) | Controls whether bots rocket-jump |
| `bot_fastchat` | `vmCvar_t` | global (extern) | Makes bots chat without delays |
| `bot_nochat` | `vmCvar_t` | global (extern) | Disables bot chat entirely |
| `bot_testrchat` | `vmCvar_t` | global (extern) | Enables random chat testing mode |
| `bot_challenge` | `vmCvar_t` | global (extern) | Increases bot difficulty/challenge |
| `ctf_redflag` | `bot_goal_t` | global (extern) | Goal descriptor for the red CTF flag |
| `ctf_blueflag` | `bot_goal_t` | global (extern) | Goal descriptor for the blue CTF flag |
| `ctf_neutralflag` | `bot_goal_t` | global (extern, `MISSIONPACK`) | Neutral flag for 1-Flag CTF |
| `redobelisk` | `bot_goal_t` | global (extern, `MISSIONPACK`) | Red obelisk goal for Overload mode |
| `blueobelisk` | `bot_goal_t` | global (extern, `MISSIONPACK`) | Blue obelisk goal for Overload mode |
| `neutralobelisk` | `bot_goal_t` | global (extern, `MISSIONPACK`) | Neutral obelisk for Harvester mode |

## Key Functions / Methods

### BotDeathmatchAI
- **Signature:** `void BotDeathmatchAI(bot_state_t *bs, float thinktime)`
- **Purpose:** Main per-frame AI entry point; drives all bot decision-making for deathmatch.
- **Inputs:** `bs` — bot state; `thinktime` — elapsed time since last think.
- **Outputs/Return:** None.
- **Side effects:** Mutates `bs` extensively; may issue movement/attack commands via botlib.
- **Calls:** Defined in `ai_dmq3.c`; calls most other functions declared in this header.
- **Notes:** Central frame-tick dispatcher for the entire bot AI.

### BotFindEnemy
- **Signature:** `int BotFindEnemy(bot_state_t *bs, int curenemy)`
- **Purpose:** Scans for a valid enemy target, setting `bs->enemy` if found.
- **Inputs:** `bs` — bot state; `curenemy` — current enemy entity number (hint).
- **Outputs/Return:** Non-zero if an enemy was found and assigned.
- **Side effects:** Modifies `bs->enemy` and related targeting fields.
- **Calls:** Not inferable from this file.
- **Notes:** Considers visibility, team membership, and entity liveness.

### BotEntityVisible
- **Signature:** `float BotEntityVisible(int viewer, vec3_t eye, vec3_t viewangles, float fov, int ent)`
- **Purpose:** Computes a visibility score `[0,1]` for an entity from a given viewpoint.
- **Inputs:** Viewer client number, eye position, view angles, FOV, target entity number.
- **Outputs/Return:** Float in `[0, 1]`; 0 = not visible, 1 = fully visible.
- **Side effects:** None.
- **Notes:** Used to qualify both enemy detection and teammate awareness.

### BotCTFSeekGoals / BotCTFRetreatGoals
- **Signature:** `void BotCTFSeekGoals(bot_state_t *bs)` / `void BotCTFRetreatGoals(bot_state_t *bs)`
- **Purpose:** Set the bot's current navigation goal based on CTF role (attack vs. defend/retreat).
- **Inputs:** `bs` — bot state with team/flag-carry status.
- **Outputs/Return:** None.
- **Side effects:** Modifies `bs` goal fields.
- **Notes:** Complementary pair; called depending on `BotWantsToRetreat` result.

### Notes (trivial helpers declared here)
- `BotIsDead`, `BotIsObserver`, `BotIntermission`, `BotInLavaOrSlime` — simple state predicate queries on `bs`.
- `EntityIsDead`, `EntityIsInvisible`, `EntityIsShooting`, `EntityHasKamikaze` — stateless predicates on `aas_entityinfo_t`.
- `BotSetupDeathmatchAI` / `BotShutdownDeathmatchAI` — one-time init/teardown with no parameters.
- `ClientName`, `EasyClientName`, `ClientSkin`, `ClientFromName`, `ClientOnSameTeamFromName` — client info lookup utilities.
- `BotCreateWayPoint`, `BotFindWayPoint`, `BotFreeWaypoints` — waypoint list lifecycle management.
- Mission Pack-only functions (`Bot1FCTF*`, `BotObelisk*`, `BotHarvester*`) are conditionally compiled under `#ifdef MISSIONPACK`.

## Control Flow Notes
This header is included by `ai_main.c` and other game-side bot files. `BotSetupDeathmatchAI` is called at bot initialization; `BotDeathmatchAI` is invoked once per server frame per bot; `BotShutdownDeathmatchAI` is called on map unload. CTF/Mission Pack goal functions are dispatched conditionally based on `gametype` at runtime.

## External Dependencies
- `bot_state_t`, `bot_waypoint_t`, `bot_goal_t`, `bot_moveresult_t`, `bot_activategoal_t` — defined in `ai_main.h` / `g_local.h`
- `aas_entityinfo_t` — defined in `be_aas.h` / botlib headers
- `vmCvar_t` — defined in `q_shared.h` / `qcommon.h`
- `vec3_t`, `qboolean` — defined in `q_shared.h`
- CTF flag constants (`CTF_FLAG_NONE/RED/BLUE`) and skin macros defined in this file; consumed by `ai_dmq3.c` and CTF-aware callers
