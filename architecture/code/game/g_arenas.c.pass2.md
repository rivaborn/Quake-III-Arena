# code/game/g_arenas.c — Enhanced Analysis

## Architectural Role

`g_arenas.c` is a specialized end-game orchestrator within the Game VM (server-side) that bridges two concerns: **statistics encoding** and **intermission scene construction**. It packages match-ending statistics into a `postgame` console command destined for the cgame and UI VMs, and simultaneously spawns a physical victory podium with player replicas that the renderer draws during the intermission sequence. This file is the authoritative source for tournament-style match conclusions in both deathmatch and team modes.

## Key Cross-References

### Incoming (who depends on this file)

| Caller | Call Site | Purpose |
|--------|-----------|---------|
| `code/game/g_main.c` | `GAME_RUN_FRAME` or intermission setup | Calls `UpdateTournamentInfo()` when match ends; calls `SpawnModelsOnVictoryPads()` during intermission init |
| `code/game/g_main.c` (server command handler) | Server console | `Svcmd_AbortPodium_f` registered as server command; stops celebration in single-player |
| `code/cgame/cg_servercmds.c` | `CG_ServerCommand("postgame ...")` | Parses the `postgame` command; populates client-side postgame state for scoreboard |
| `code/q3_ui/` / `code/ui/` | Postgame menu/scoreboard | Consumes postgame stats from cgame to render leaderboard UI |

### Outgoing (what this file depends on)

| Subsystem | Exports Used | Purpose |
|-----------|--------------|---------|
| `g_local.h` | `gentity_t`, `level_locals_t`, `playerState_t`, `gClientStatus_t` | Entity pool, client state, rank/score tracking |
| `code/game/g_main.c` | `CalculateRanks()` | Compute and store final rank order in `level.sortedClients[]` |
| `code/qcommon` | `trap_SendConsoleCommand()`, `trap_LinkEntity()`, `trap_Cvar_VariableIntegerValue()` | Command broadcast, entity physics registration, cvar access (g_podiumDist, g_podiumDrop) |
| `code/qcommon` / `q_shared.c` | Math: `VectorSubtract()`, `vectoangles()`, `AngleVectors()`, `VectorMA()`, `vectoyaw()` | 3D transforms for podium/replica positioning |
| Entity helpers | `G_Spawn()`, `G_SetOrigin()`, `G_ModelIndex()`, `G_AddEvent()`, `G_Printf()` | Entity lifecycle, model lookup, event firing |
| Global `level` state | `level.intermission_origin`, `level.intermission_angle`, `level.time`, `level.sortedClients[]`, `level.clients[]` | Intermission camera location, time base, ranked client list |

## Design Patterns & Rationale

### 1. **Stat Encoding as Broadcast Commands**
The `postgame` command is a text-formatted console command enqueued via `trap_SendConsoleCommand(EXEC_APPEND, ...)` rather than packed into a snapshot. This design choice prioritizes **debuggability** (human-readable format) and **decoupling** (cgame can parse it without tight binary codec contracts). Modern engines use binary serialization, but Q3's approach is appropriate for a 2005 codebase where text parsing was acceptable.

### 2. **Conditional Compilation for Game Variants**
Lines with `#ifdef MISSIONPACK` branch the postgame payload to include team scores, captures, and defend/assist counts. This is a **compile-time conditional**, not a runtime check — the binary is either base-Q3A or MissionPack, never both. This avoids runtime overhead but requires separate builds.

### 3. **Entity-Based Scene Construction**
The victory podium is a full gentity with physics properties (linked, clipped, content flags). The three player replicas are also gentities, but **static snapshots** (no think, no movement code). This economizes on entity slots versus spawning full character controllers, yet maintains compatibility with the renderer's entity system. The podium itself has a think function, making it a **mobile reference frame** for the replicas.

### 4. **Continuous Reorientation via Think Chain**
`PodiumPlacementThink` runs every 100ms to recompute the podium and replicas' positions based on the intermission camera angle. This allows **dynamic cvar tuning** (g_podiumDist, g_podiumDrop) without rebuilding the map, and ensures the scene always faces the camera. The periodic update interval is hardcoded (100ms) and assumes a fixed server tick rate.

### 5. **Animation State Machine with Fixed Timers**
`CelebrateStart` schedules `CelebrateStop` via a closure-style think chain with a fixed `TIMER_GESTURE` constant (34×66+50 ms ≈ 2244 ms). This pattern is idiomatic to Q3 for simple state transitions without a full FSM framework.

## Data Flow Through This File

```
[Match End] 
  ↓ 
UpdateTournamentInfo()
  ├─ CalculateRanks()  [compute sorted ranking]
  ├─ Scan g_entities[] for non-bot player
  ├─ Com_sprintf() format "postgame <client_id> <accuracy> <rank1_id> ... "
  └─ trap_SendConsoleCommand(EXEC_APPEND, msg)
        ↓ (queued on server)
    [Server Frame Tick]
      ├─ Execute console command
      ├─ Broadcast to all clients
      └─ cgame VM's CG_ServerCommand() receives & parses
          ├─ Populates cg.postgameStats or similar
          └─ [UI VM renders postgame menu]

[Intermission Setup]
  ↓ 
SpawnModelsOnVictoryPads()
  ├─ SpawnPodium()
  │   ├─ G_Spawn(), G_SetOrigin(), trap_LinkEntity()
  │   └─ Schedule PodiumPlacementThink() every 100ms
  ├─ For rank 0, 1, 2:
  │   ├─ SpawnModelOnVictoryPad() → static body gentity
  │   ├─ Copy player state snapshot (ent->s → body->s)
  │   └─ Apply rank-specific animation (CelebrateStart for rank 0)
  └─ podium1/2/3 global pointers set

[Intermission Frames]
  ├─ PodiumPlacementThink() (every 100ms)
  │   └─ Reorient podium & replicas toward level.intermission_angle
  ├─ CelebrateStart() → CelebrateStop() (after ~2.2s)
  │   └─ Toggle torso animation on rank 0 replica
  └─ Renderer draws podium + replicas as normal entities
```

## Learning Notes

1. **Snapshot vs. Streaming Commands**: The postgame stats use a console command (streamed), not an entity snapshot (delta-compressed). This reflects Q3's philosophy of separating **game state** (snapshots, high-frequency updates) from **administrative events** (commands, low-frequency state transitions).

2. **Animation Toggle Encoding**: The `ANIM_TOGGLEBIT` XOR trick (`player->s.torsoAnim = ((player->s.torsoAnim & ANIM_TOGGLEBIT) ^ ANIM_TOGGLEBIT) | TORSO_GESTURE`) encodes both the animation type and a toggle bit in a single 32-bit field. Modern engines use separate fields; Q3 optimizes for bandwidth.

3. **Weapon-Centric Design**: Even cosmetic replica models check `body->s.weapon == WP_GAUNTLET` to apply weapon-specific animations. This shows Q3's pervasive weapon-driven architecture.

4. **Cvar-Driven Authoring**: The podium position is tunable via `g_podiumDist` and `g_podiumDrop` cvars, read **every frame** from `PodiumPlacementThink`. This allows designers to adjust the scene without recompilation, but incurs per-frame cvar lookup overhead.

5. **Deterministic Ordering via Sorting**: The victory podium ranks players using `level.sortedClients[0/1/2]`, which are populated by `CalculateRanks()`. This ensures consistent ordering across servers and prevents race conditions.

6. **Spectator Special Case**: Players in `TEAM_SPECTATOR` are handled with zeroed stats in the postgame command, preventing them from appearing on the leaderboard but still sending the command so the client UI doesn't crash.

## Potential Issues

1. **Buffer Overflow on Large Leaderboards** (line 143–148): The loop appending per-client ranks checks `if (msglen + buflen + 1 >= sizeof(msg))` and breaks, but no error is logged. On very large servers (e.g., 64+ players), the postgame command is silently truncated without warning. The first few clients' stats are always included, but the tail of the leaderboard is lost.

2. **Spectator Loop Inefficiency** (line 52–62): If all non-bots are spectators, the loop scans every client to find a non-bot, then early-returns. On a 64-client server, this is 64 iterations for a NULL return. Using `level.numNonSpectatorClients` would be faster, but the code doesn't.

3. **Missing NULL Guard after `G_Spawn()`** (line 181–183): If `SpawnModelOnVictoryPad()` returns NULL due to entity pool exhaustion, the callers in `SpawnModelsOnVictoryPads()` still execute the `if (player)` block. However, if all three fail, the podium exists but is empty — not a crash, but visually broken.

4. **Hardcoded Animation Timing**: `TIMER_GESTURE = 34*66+50` is burned into the code. If animation frame rates or network ticks change, this tuning constant won't adapt. The cgame VM may play the gesture animation for a different duration, causing desynchronization.

5. **Cvar Lookup Overhead**: `PodiumPlacementThink()` calls `trap_Cvar_VariableIntegerValue()` twice per tick (lines 211–212). This is a hashtable lookup; caching the value or using a callback would be more efficient. For a smooth 60 Hz client, this is 6–12 cvar lookups per second during intermission.

6. **Think Interval Hard-coded to 100ms**: The podium reorients every 100ms (line 210). If the server runs at 20 Hz (50ms ticks) or 50 Hz (20ms ticks), the podium will visibly jitter or snap-turn. Modern engines decouple visual updates from server frame rate.

## Architectural Dependencies Summary

- **Tightly coupled to**: `g_local.h`, `g_main.c` (CalculateRanks), game VM's entity lifecycle (`G_Spawn`, etc.)
- **Loosely coupled to**: cgame VM (consumes postgame command via trap layer), UI VMs (display the stats)
- **No direct dependency on**: renderer, physics engine, botlib, or collision system (all indirect via entity/trap system)
- **No bidirectional calls**: This file only calls outward; nothing calls back into it except entry points (`UpdateTournamentInfo`, `SpawnModelsOnVictoryPads`, `Svcmd_AbortPodium_f`)
