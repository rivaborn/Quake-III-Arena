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

## Key Types / Data Structures
None (all types are defined in `g_local.h`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `podium1` | `gentity_t *` | global | Gentity for the 1st-place player model on the podium |
| `podium2` | `gentity_t *` | global | Gentity for the 2nd-place player model |
| `podium3` | `gentity_t *` | global | Gentity for the 3rd-place player model |
| `offsetFirst` | `vec3_t` | file-static | Positional offset for 1st-place model relative to podium |
| `offsetSecond` | `vec3_t` | file-static | Positional offset for 2nd-place model |
| `offsetThird` | `vec3_t` | file-static | Positional offset for 3rd-place model |

## Key Functions / Methods

### UpdateTournamentInfo
- **Signature:** `void UpdateTournamentInfo( void )`
- **Purpose:** Builds and dispatches the `postgame` console command encoding end-of-match stats for the scoreboard UI.
- **Inputs:** None (reads `level`, `g_entities`, `g_gametype`)
- **Outputs/Return:** void
- **Side effects:** Calls `CalculateRanks()`; calls `trap_SendConsoleCommand(EXEC_APPEND, msg)` to enqueue the postgame command on the server.
- **Calls:** `CalculateRanks`, `Com_sprintf`, `strlen`, `strcat`, `trap_SendConsoleCommand`
- **Notes:** Handles spectating players by emitting zeroed stats. MISSIONPACK build adds team scores, defend/assist/capture counts, and a `won` flag. The `perfect` flag requires rank 0 and zero deaths (or team win with zero deaths in MISSIONPACK).

### SpawnModelOnVictoryPad
- **Signature:** `static gentity_t *SpawnModelOnVictoryPad( gentity_t *pad, vec3_t offset, gentity_t *ent, int place )`
- **Purpose:** Spawns a stationary cosmetic copy of a player entity positioned on the victory podium.
- **Inputs:** `pad` – podium anchor entity; `offset` – 3D offset from podium in local frame; `ent` – source player entity; `place` – rank value stored on body.
- **Outputs/Return:** Pointer to the new body `gentity_t`, or `NULL` on allocation failure.
- **Side effects:** Calls `G_Spawn`, `G_SetOrigin`, `trap_LinkEntity`; writes to `g_entities` pool.
- **Calls:** `G_Spawn`, `G_Printf`, `VectorSubtract`, `vectoangles`, `AngleVectors`, `VectorMA`, `G_SetOrigin`, `trap_LinkEntity`
- **Notes:** Forces `ET_PLAYER` type, clears powerups/events/loopSound; defaults weapon to `WP_MACHINEGUN` if none; uses `TORSO_STAND2` for gauntlet wielders.

### CelebrateStart / CelebrateStop
- **Signature:** `static void CelebrateStart( gentity_t *player )` / `static void CelebrateStop( gentity_t *player )`
- **Purpose:** Toggle the winner's torso gesture animation on and off via the entity think chain.
- **Side effects:** Modifies `player->s.torsoAnim`; schedules `CelebrateStop` via `nextthink`/`think`; fires `EV_TAUNT` event via `G_AddEvent`.
- **Notes:** `TIMER_GESTURE` = 34×66+50 ms controls how long the gesture plays.

### PodiumPlacementThink
- **Signature:** `static void PodiumPlacementThink( gentity_t *podium )`
- **Purpose:** Per-frame (100 ms) think callback that repositions the podium and all three player models to face the intermission camera.
- **Side effects:** Calls `G_SetOrigin` on the podium and each of `podium1/2/3`; reads `g_podiumDist` and `g_podiumDrop` cvars each tick.
- **Notes:** Reschedules itself every 100 ms; safe if any of `podium1/2/3` is NULL.

### SpawnPodium
- **Signature:** `static gentity_t *SpawnPodium( void )`
- **Purpose:** Allocates and initializes the physical podium model entity.
- **Side effects:** Calls `G_Spawn`, `G_ModelIndex`, `G_SetOrigin`, `trap_LinkEntity`; sets `think = PodiumPlacementThink`.
- **Calls:** `G_Spawn`, `G_ModelIndex`, `AngleVectors`, `VectorMA`, `G_SetOrigin`, `VectorSubtract`, `vectoyaw`, `trap_LinkEntity`

### SpawnModelsOnVictoryPads
- **Signature:** `void SpawnModelsOnVictoryPads( void )`
- **Purpose:** Public entry point that spawns the podium and up to three player models; wires up the celebration think for rank 1.
- **Side effects:** Resets `podium1/2/3` globals; calls `SpawnPodium` and up to three `SpawnModelOnVictoryPad` calls.

### Svcmd_AbortPodium_f
- **Signature:** `void Svcmd_AbortPodium_f( void )`
- **Purpose:** Server command handler to immediately stop the celebration animation; only active in `GT_SINGLE_PLAYER`.
- **Side effects:** Sets `podium1->nextthink = level.time` and `think = CelebrateStop`.

## Control Flow Notes
- `UpdateTournamentInfo` is called from `g_main.c` when the match ends to push the postgame stats.
- `SpawnModelsOnVictoryPads` is called during intermission setup (after `BeginIntermission`) to populate the victory scene.
- `PodiumPlacementThink` runs every server frame (100 ms tick) during the intermission via the entity think system.

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `level` (`level_locals_t`), `g_entities[]`, `g_gametype`; trap functions (`trap_SendConsoleCommand`, `trap_LinkEntity`, `trap_Cvar_VariableIntegerValue`); math utilities (`AngleVectors`, `VectorMA`, `vectoangles`, `vectoyaw`); entity helpers (`G_Spawn`, `G_SetOrigin`, `G_ModelIndex`, `G_AddEvent`, `G_Printf`); `CalculateRanks`; `SP_PODIUM_MODEL` (defined in `g_local.h`).
