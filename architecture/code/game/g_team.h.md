# code/game/g_team.h

## File Purpose
Header file for the Quake III Arena team-based game mode (CTF and Missionpack variants). It defines scoring constants for Capture the Flag mechanics and declares the public interface for team logic used by the server-side game module.

## Core Responsibilities
- Declares CTF scoring bonus constants, conditionally compiled for MISSIONPACK vs. base Q3A balancing
- Declares geometric radius and timing constants for proximity-based bonus logic
- Declares grapple hook physics constants
- Exposes the public function interface for all team/CTF game logic to the rest of the game module

## Key Types / Data Structures
None. (Types used — `gentity_t`, `vec3_t`, `team_t`, `qboolean` — are defined in `g_local.h` / `q_shared.h`.)

## Global / File-Static State
None declared in this header.

## Key Functions / Methods

### OtherTeam / TeamName / OtherTeamName / TeamColorString
- Signature: `int OtherTeam(int team)` / `const char *TeamName(int team)` / etc.
- Purpose: Utility lookups — return the opposing team index or human-readable team name/color string.
- Inputs: `team` — integer team index.
- Outputs/Return: Opposing team index or string constant.
- Side effects: None (pure lookups).
- Calls: Not inferable from this file.
- Notes: Trivial helpers surfaced for use across multiple game source files.

### AddTeamScore
- Signature: `void AddTeamScore(vec3_t origin, int team, int score)`
- Purpose: Awards score to a team, likely triggering HUD/world feedback at `origin`.
- Inputs: World-space position, team index, score delta.
- Outputs/Return: void.
- Side effects: Modifies team score state; may emit events or prints.
- Calls: Not inferable from this file.

### Team_FragBonuses
- Signature: `void Team_FragBonuses(gentity_t *targ, gentity_t *inflictor, gentity_t *attacker)`
- Purpose: Evaluates and awards proximity/carrier-protection scoring bonuses when a frag occurs in a team game.
- Inputs: The killed entity, the damage source entity, and the attacking player entity.
- Outputs/Return: void.
- Side effects: Modifies player scores; uses `CTF_*_BONUS` and `CTF_*_RADIUS`/`TIMEOUT` constants.
- Calls: Not inferable from this file.
- Notes: Central to the CTF bonus scoring system — checks carrier proximity, flag defense, and recent hurt-carrier events.

### Team_CheckHurtCarrier
- Signature: `void Team_CheckHurtCarrier(gentity_t *targ, gentity_t *attacker)`
- Purpose: Records that an attacker has recently damaged a flag carrier, enabling `CTF_CARRIER_DANGER_PROTECT_BONUS` within `CTF_CARRIER_DANGER_PROTECT_TIMEOUT`.
- Inputs: Damaged entity (flag carrier), attacking entity.
- Side effects: Writes timestamp/attacker data to game entity state.

### Team_InitGame / Team_ReturnFlag / Team_FreeEntity
- Purpose: Lifecycle management — initialize team state at game start, return a flag to base (auto-return or manual), and clean up team-associated entities on removal.
- Notes: `Team_ReturnFlag` is triggered by the `CTF_FLAG_RETURN_TIME` (40000 ms) timer.

### SelectCTFSpawnPoint
- Signature: `gentity_t *SelectCTFSpawnPoint(team_t team, int teamstate, vec3_t origin, vec3_t angles)`
- Purpose: Selects an appropriate spawn point for a player given their team and the current game state.
- Outputs/Return: Pointer to a spawn point `gentity_t`; writes position/angles into out-params.

### Team_GetLocation / Team_GetLocationMsg / TeamplayInfoMessage / CheckTeamStatus
- Purpose: Location reporting utilities — resolve a player's nearest named location entity, format a location string, and broadcast team status messages to teammates.
- Notes: Used for team HUD and voice/chat location tagging.

### Pickup_Team
- Signature: `int Pickup_Team(gentity_t *ent, gentity_t *other)`
- Purpose: Handles the event of a player picking up a team item (e.g., flag).
- Outputs/Return: Integer — likely a respawn/hold timer value consistent with Q3A item pickup convention.

### Team_DroppedFlagThink
- Signature: `void Team_DroppedFlagThink(gentity_t *ent)`
- Purpose: Think callback for a dropped flag entity; drives auto-return countdown using `CTF_FLAG_RETURN_TIME`.
- Side effects: May call `Team_ReturnFlag` when the timer expires.

## Control Flow Notes
This header is included by `g_team.c` (implementation) and any other game-side `.c` file that needs to call team/CTF logic (e.g., `g_combat.c` for frag bonuses, `g_items.c` for flag pickup). It is entirely server-side game module code; no client or renderer dependency.

## External Dependencies
- `gentity_t`, `team_t`, `vec3_t`, `qboolean` — defined in `g_local.h` / `q_shared.h`
- `MISSIONPACK` — preprocessor define controlling two distinct scoring balance sets; defined at build time
- All function bodies defined in `g_team.c`
