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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `grank_status_t` | typedef (enum, defined elsewhere) | Represents a client's current ranking auth status (QGR_STATUS_*) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `last_framenum` | `static int` | file-static (inside `G_RankDamage`) | Deduplicates multi-pellet shotgun hits within a single frame |
| `last_self` | `static int` | file-static | Part of shotgun dedup key |
| `last_attacker` | `static int` | file-static | Part of shotgun dedup key |
| `last_means_of_death` | `static int` | file-static | Part of shotgun dedup key |

## Key Functions / Methods

### G_RankRunFrame
- **Signature:** `void G_RankRunFrame()`
- **Purpose:** Per-frame tick for the rankings subsystem; initializes if needed, polls the service, manages client auth states, and updates `PERS_MATCH_TIME`.
- **Inputs:** None (reads `level`, `g_entities` globals)
- **Outputs/Return:** void
- **Side effects:** May kick bots via `trap_SendConsoleCommand`; sends `rank_status`/`rank_menu` server commands to clients; may force `timelimit` cvar to 1000; spawns/moves clients between spectator and active teams; calls `trap_RankReportInt` for `QGR_KEY_PLAYED_WITH`; writes `ps.persistant[PERS_MATCH_TIME]`
- **Calls:** `trap_RankCheckInit`, `trap_RankBegin`, `trap_RankPoll`, `trap_RankActive`, `trap_RankUserStatus`, `trap_RankUserReset`, `trap_RankReportInt`, `trap_SendConsoleCommand`, `trap_SendServerCommand`, `trap_Cvar_Set`, `ClientSpawn`, `SetTeam`, `DeathmatchScoreboardMessage`
- **Notes:** Must be called every server frame. Bot entities are kicked when a ranked game is active. The dead-code block `if (i == 0) { int j = 0; }` is vestigial debug code.

### G_RankFireWeapon
- **Signature:** `void G_RankFireWeapon( int self, int weapon )`
- **Purpose:** Reports a weapon discharge event to the rankings service; skipped during warmup; gauntlet is excluded (it only counts on hit).
- **Inputs:** `self` — client entity index; `weapon` — `WP_*` enum value
- **Outputs/Return:** void
- **Side effects:** Calls `trap_RankReportInt` for `QGR_KEY_SHOT_FIRED` (general) and the weapon-specific key
- **Calls:** `trap_RankReportInt`
- **Notes:** Called from `g_weapon.c` at fire time, not on hit.

### G_RankDamage
- **Signature:** `void G_RankDamage( int self, int attacker, int damage, int means_of_death )`
- **Purpose:** Reports a damage event (hit taken/given, damage taken/given, splash, friendly fire) to the rankings service.
- **Inputs:** `self` — victim entity index; `attacker` — attacker entity index or `ENTITYNUM_WORLD`; `damage` — HP lost; `means_of_death` — `MOD_*` enum
- **Outputs/Return:** void
- **Side effects:** Updates the four `static` dedup variables; calls `trap_RankReportInt` up to ~12 times per unique hit event; gauntlet fire is counted here since it only fires on contact
- **Calls:** `trap_RankReportInt`, `OnSameTeam`
- **Notes:** The four `static` locals deduplicate shotgun pellets — only the first pellet in a frame/target combination counts as a new hit. Hazard damage (water/slime/lava/crush/telefrag/falling/suicide/trigger) is silently skipped for damage tracking (deaths are reported separately in `G_RankPlayerDie`). Guard against non-client `attacker` indices (e.g., grenade-shooter proxy at index 245).

### G_RankPlayerDie
- **Signature:** `void G_RankPlayerDie( int self, int attacker, int means_of_death )`
- **Purpose:** Reports a player death, classified into hazard kill, suicide, or frag with per-weapon detail.
- **Inputs:** `self` — victim; `attacker` — killer or `ENTITYNUM_WORLD` or equals `self`
- **Outputs/Return:** void
- **Side effects:** Calls `trap_RankReportInt` twice per death (general + specific key)
- **Calls:** `trap_RankReportInt`

### G_RankWeaponTime
- **Signature:** `void G_RankWeaponTime( int self, int weapon )`
- **Purpose:** Reports seconds spent holding a weapon before switching; resets `client->weapon_change_time`.
- **Inputs:** `self` — client index; `weapon` — weapon being switched away from
- **Outputs/Return:** void
- **Side effects:** Writes `client->weapon_change_time`; calls `trap_RankReportInt`
- **Calls:** `trap_RankReportInt`
- **Notes:** Called on weapon change events.

### G_RankGameOver
- **Signature:** `void G_RankGameOver( void )`
- **Purpose:** Flushes all active clients' match ratings then submits session-level metadata (hostname, map, mod, gametype, limits, server config, version).
- **Inputs:** None (reads cvars and `level`)
- **Outputs/Return:** void
- **Side effects:** Calls `G_RankClientDisconnect` for each active client; calls `trap_RankReportStr`/`trap_RankReportInt` ~10 times for session metadata
- **Calls:** `trap_RankUserStatus`, `G_RankClientDisconnect`, `trap_Cvar_VariableStringBuffer`, `trap_Cvar_VariableIntegerValue`, `trap_RankReportStr`, `trap_RankReportInt`

### G_RankClientDisconnect
- **Signature:** `void G_RankClientDisconnect( int self )`
- **Purpose:** Computes and submits a match rating for a disconnecting/finishing client; rating is 0 if they played less than 60 seconds.
- **Inputs:** `self` — client index
- **Outputs/Return:** void
- **Side effects:** Calls `trap_RankReportInt` for `QGR_KEY_MATCH_RATING`
- **Notes:** Also called by `G_RankGameOver` for all still-active clients at match end.

- **Notes on trivial helpers:** `G_RankPickupWeapon`, `G_RankPickupAmmo`, `G_RankPickupHealth`, `G_RankPickupArmor`, `G_RankPickupPowerup`, `G_RankPickupHoldable`, `G_RankUseHoldable`, `G_RankReward`, `G_RankCapture`, `G_RankUserTeamName` — all follow the same pattern: early-out during warmup, then one or two `trap_RankReportInt`/`trap_RankReportStr` calls dispatched on the item/event type.

## Control Flow Notes
`G_RankRunFrame` is called from `G_RunFrame` (game frame loop) every server frame. All other `G_Rank*` functions are event-driven callbacks invoked from the appropriate game subsystems (`g_weapon.c`, `g_combat.c`, `g_items.c`, `g_client.c`, `g_main.c`) at the point the game event occurs. `G_RankGameOver` is called at intermission/match-end. All functions are no-ops during warmup (`level.warmupTime != 0`).

## External Dependencies
- **Includes:** `g_local.h` (game entity/client types, level globals, all trap declarations), `g_rankings.h` (QGR_KEY_* constants, `GR_GAMEKEY`)
- **Defined elsewhere:** `trap_RankCheckInit`, `trap_RankBegin`, `trap_RankPoll`, `trap_RankActive`, `trap_RankUserStatus`, `trap_RankUserReset`, `trap_RankReportInt`, `trap_RankReportStr` — ranking system trap calls into the engine/VM syscall layer; `level` (`level_locals_t`), `g_entities[]` — game globals; `ClientSpawn`, `SetTeam`, `DeathmatchScoreboardMessage`, `OnSameTeam` — other game module functions; `GR_GAMEKEY` — game-key constant (defined elsewhere, not in the provided headers)
