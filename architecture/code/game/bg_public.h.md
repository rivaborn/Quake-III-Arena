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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `pmove_t` | struct | Full input/output context for a single physics simulation step; bridges engine, game, and cgame |
| `gitem_t` | struct | Describes a spawnable/pickup item (classname, models, sounds, quantity, type, tag) |
| `animation_t` | struct | Describes a skeletal animation clip (frames, loop, lerp timing, reverse/flipflop flags) |
| `gametype_t` | enum | All supported game modes (FFA, Tournament, CTF, Obelisk, Harvester, etc.) |
| `pmtype_t` | enum | Player movement mode (normal, noclip, spectator, dead, freeze, intermission) |
| `weaponstate_t` | enum | Weapon FSM states (ready, raising, dropping, firing) |
| `weapon_t` | enum | All weapon identifiers including MissionPack weapons under `#ifdef MISSIONPACK` |
| `powerup_t` | enum | All powerup identifiers |
| `holdable_t` | enum | All holdable item identifiers |
| `entity_event_t` | enum | All entity-relative game events (footsteps, hits, deaths, powerups, taunts, etc.) |
| `entityType_t` | enum | Entity type classifications used in `entityState_t->eType` |
| `animNumber_t` | enum | All animation clip indices for player models |
| `meansOfDeath_t` | enum | All kill causes for obituary and scoring logic |
| `itemType_t` | enum | Item category flags driving pickup effects |
| `statIndex_t` | enum | Indices into `playerState_t->stats[]` |
| `persEnum_t` | enum | Indices into `playerState_t->persistant[]` |
| `team_t` | enum | Team membership (free, red, blue, spectator) |
| `teamtask_t` | enum | Bot/player team role assignments |
| `global_team_sound_t` | enum | Global team audio event identifiers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `bg_itemlist` | `gitem_t[]` | global (extern) | Master item definition table; defined in `bg_misc.c`, shared across game and cgame |
| `bg_numItems` | `int` | global (extern) | Count of entries in `bg_itemlist` |

## Key Functions / Methods

### Pmove
- **Signature:** `void Pmove(pmove_t *pmove)`
- **Purpose:** Execute one full player physics simulation step (movement, collisions, water, weapons, events).
- **Inputs:** `pmove` — fully populated `pmove_t` with current `playerState_t`, `usercmd_t`, trace callbacks, and config flags.
- **Outputs/Return:** Mutates `pmove->ps` (new player state) and populates `numtouch`/`touchents`, `watertype`, `waterlevel`, `xyspeed`.
- **Side effects:** Calls `pmove->trace` and `pmove->pointcontents` callbacks; fires predictable events into player state.
- **Calls:** Defined in `bg_pmove.c`; internally calls slide-move, water-move, weapon-state helpers.
- **Notes:** Runs identically on server (authoritative) and client (prediction); divergence causes visual correction snaps.

### PM_UpdateViewAngles
- **Signature:** `void PM_UpdateViewAngles(playerState_t *ps, const usercmd_t *cmd)`
- **Purpose:** Lightweight angle update without a full pmove (used when a full physics step is skipped).
- **Inputs:** Current `playerState_t`, incoming `usercmd_t`.
- **Outputs/Return:** Mutates `ps->viewangles`.
- **Side effects:** None beyond the player state angle fields.
- **Calls:** Defined in `bg_pmove.c`.

### BG_EvaluateTrajectory / BG_EvaluateTrajectoryDelta
- **Signature:** `void BG_EvaluateTrajectory(const trajectory_t *tr, int atTime, vec3_t result)` / delta variant
- **Purpose:** Compute entity position (or velocity) along a stored trajectory at a given timestamp; used for missile and mover prediction on both client and server.

### BG_PlayerStateToEntityState / ExtraPolate variant
- **Signature:** `void BG_PlayerStateToEntityState(playerState_t *ps, entityState_t *s, qboolean snap)`
- **Purpose:** Convert authoritative player state into a network-transmittable entity state snapshot.

### BG_AddPredictableEventToPlayerstate
- **Purpose:** Enqueue a game event in `ps->events[]` in a way that both server and client can predict without duplication.

### BG_FindItem / BG_FindItemForWeapon / BG_FindItemForPowerup / BG_FindItemForHoldable
- **Purpose:** Lookup helpers that search `bg_itemlist` by name, weapon enum, powerup enum, or holdable enum.
- **Notes:** `ITEM_INDEX(x)` macro converts a `gitem_t*` to its integer index via pointer arithmetic.

### BG_CanItemBeGrabbed
- **Signature:** `qboolean BG_CanItemBeGrabbed(int gametype, const entityState_t *ent, const playerState_t *ps)`
- **Purpose:** Determine whether a player can pick up an item given current game type and player state (e.g. armor/health cap checks).

### BG_PlayerTouchesItem
- **Purpose:** Spatial check — returns true if the player's bounding box at `atTime` overlaps an item entity.

## Control Flow Notes
This is a **header only** — no runtime execution occurs here. It is `#include`d by:
- `game/` (server VM) — authoritative pmove and game logic.
- `cgame/` (client VM) — client-side prediction and rendering queries.
- `bg_*.c` files implement all declared `BG_*` and `Pmove` functions; they compile into **both** VMs to guarantee identical behavior.

## External Dependencies
- `q_shared.h` — `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `vec3_t`, `trace_t`, `qboolean`, `CONTENTS_*`, `MAX_*` constants, `CS_SERVERINFO`/`CS_SYSTEMINFO`
- `MISSIONPACK` preprocessor define — gates additional weapons (`WP_NAILGUN`, `WP_PROX_LAUNCHER`, `WP_CHAINGUN`), powerups, means of death, and entity flags for the Team Arena expansion
- All `BG_*` function bodies defined in `bg_misc.c`, `bg_pmove.c`, `bg_slidemove.c`, `bg_lib.c`
