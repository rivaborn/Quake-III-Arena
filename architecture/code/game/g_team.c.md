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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `teamgame_t` | struct | Per-match team game state: flag statuses, capture timestamps, obelisk attack times |
| `flagStatus_t` | typedef (enum, defined in `g_team.h`) | Enum for flag state: `FLAG_ATBASE`, `FLAG_TAKEN`, `FLAG_DROPPED`, `FLAG_TAKEN_RED`, `FLAG_TAKEN_BLUE` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `teamgame` | `teamgame_t` | global | Single instance holding all runtime team game state |
| `neutralObelisk` | `gentity_t *` | global | Pointer to the neutral obelisk entity (Harvester mode) |
| `ctfFlagStatusRemap[]` | `static char[]` | static | Maps `flagStatus_t` values to config string characters for GT_CTF |
| `oneFlagStatusRemap[]` | `static char[]` | static | Maps `flagStatus_t` values to config string characters for GT_1FCTF |

## Key Functions / Methods

### Team_InitGame
- Signature: `void Team_InitGame(void)`
- Purpose: Resets `teamgame` state and initializes flag statuses for the current gametype.
- Inputs: None (reads `g_gametype.integer`)
- Outputs/Return: void
- Side effects: Zeroes `teamgame`, calls `Team_SetFlagStatus` to push initial config strings.
- Calls: `memset`, `Team_SetFlagStatus`
- Notes: Called at map start. GT_CTF forces a dirty status to trigger immediate config string update.

### Team_SetFlagStatus
- Signature: `void Team_SetFlagStatus(int team, flagStatus_t status)`
- Purpose: Updates the cached flag status for a team and, if changed, encodes and pushes the new status to the config string `CS_FLAGSTATUS`.
- Inputs: `team` (TEAM_RED/BLUE/FREE), `status`
- Outputs/Return: void
- Side effects: Writes `CS_FLAGSTATUS` config string via `trap_SetConfigstring`.
- Calls: `trap_SetConfigstring`

### Team_FragBonuses
- Signature: `void Team_FragBonuses(gentity_t *targ, gentity_t *inflictor, gentity_t *attacker)`
- Purpose: Evaluates and awards bonus scores for killing the enemy flag carrier, defending the carrier, and defending the base flag.
- Inputs: Target, inflictor, attacker entities.
- Outputs/Return: void
- Side effects: Calls `AddScore`, sets `eFlags` award bits, sets `rewardTime`, prints messages via `PrintMsg`.
- Calls: `OnSameTeam`, `OtherTeam`, `AddScore`, `PrintMsg`, `G_Find`, `VectorSubtract`, `VectorLength`, `trap_InPVS`
- Notes: Bonuses are non-cumulative; checked in priority order. Contains a latent bug: both `v1` subtraction calls in the carrier-protect check use `v1` instead of `v2` for the attacker distance.

### Team_TouchOurFlag
- Signature: `int Team_TouchOurFlag(gentity_t *ent, gentity_t *other, int team)`
- Purpose: Handles a player touching their own team's flag entity — either returning a dropped flag or scoring a capture if carrying the enemy flag.
- Inputs: Flag entity, touching player entity, team id.
- Outputs/Return: 0 (do not auto-respawn flag)
- Side effects: `AddScore`, `AddTeamScore`, `Team_ReturnFlagSound`, `Team_ResetFlags`, `CalculateRanks`, sets award flags.
- Calls: `PrintMsg`, `AddScore`, `AddTeamScore`, `Team_ForceGesture`, `Team_CaptureFlagSound`, `Team_ResetFlags`, `CalculateRanks`

### Team_TouchEnemyFlag
- Signature: `int Team_TouchEnemyFlag(gentity_t *ent, gentity_t *other, int team)`
- Purpose: Handles a player picking up the enemy (or neutral) flag.
- Inputs: Flag entity, touching player, owning team of the flag.
- Outputs/Return: -1 (delete if dropped, do not respawn)
- Side effects: Sets powerup on player, calls `Team_SetFlagStatus`, `AddScore`, `Team_TakeFlagSound`.

### Pickup_Team
- Signature: `int Pickup_Team(gentity_t *ent, gentity_t *other)`
- Purpose: Entry point called by the item system when a player touches a team item; dispatches to `Team_TouchOurFlag` or `Team_TouchEnemyFlag` based on gametype and team ownership.
- Inputs: Item entity, picking-up player.
- Outputs/Return: Respawn time hint (0 or -1).
- Calls: `Team_TouchOurFlag`, `Team_TouchEnemyFlag`, `G_FreeEntity`, `PrintMsg`

### CheckTeamStatus
- Signature: `void CheckTeamStatus(void)`
- Purpose: Periodically (every `TEAM_LOCATION_UPDATE_TIME` ms) updates each team player's location index and sends team overlay info messages.
- Inputs: None (reads `level`)
- Side effects: Writes `pers.teamState.location` per client; calls `trap_SendServerCommand` via `TeamplayInfoMessage`.
- Calls: `Team_GetLocation`, `TeamplayInfoMessage`
- Notes: Called each server frame from `g_active.c`.

### SpawnObelisk *(MISSIONPACK)*
- Signature: `gentity_t *SpawnObelisk(vec3_t origin, int team, int spawnflags)`
- Purpose: Creates and configures an obelisk physics/trigger entity, drops it to the floor.
- Side effects: Allocates entity, calls `trap_Trace`, `trap_LinkEntity`.

### CheckObeliskAttack *(MISSIONPACK)*
- Signature: `qboolean CheckObeliskAttack(gentity_t *obelisk, gentity_t *attacker)`
- Purpose: Called before damage is applied to determine if the obelisk should be hurt; suppresses friendly fire and rate-limits attack sound events.
- Outputs/Return: `qtrue` to block damage (friendly or not an obelisk), `qfalse` to allow.
- Side effects: May spawn `EV_GLOBAL_TEAM_SOUND` temp entity and update `teamgame.*ObeliskAttackedTime`.

## Control Flow Notes
- `Team_InitGame` is called during map initialization.
- `Team_FragBonuses` and `Team_CheckHurtCarrier` are called from `g_combat.c` on kill/damage events.
- `Pickup_Team` is called from `g_items.c` touch dispatch.
- `Team_DroppedFlagThink` is registered as a `think` callback on dropped flag entities (set in `g_items.c`/`LaunchItem`).
- `CheckTeamStatus` is called every server frame from `g_active.c`.
- Obelisk `think`/`pain`/`die`/`touch` callbacks are registered at spawn time.

## External Dependencies
- `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, `g_team.h`)
- **Defined elsewhere:** `AddScore`, `CalculateRanks`, `G_Find`, `G_TempEntity`, `G_Spawn`, `G_FreeEntity`, `G_SetOrigin`, `RespawnItem`, `SelectSpawnPoint`, `SpotWouldTelefrag`, `trap_SetConfigstring`, `trap_SendServerCommand`, `trap_InPVS`, `trap_Trace`, `trap_LinkEntity`, `level` (global), `g_entities` (global), all `g_obelisk*` cvars.
