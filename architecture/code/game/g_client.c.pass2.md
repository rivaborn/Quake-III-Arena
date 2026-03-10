# code/game/g_client.c — Enhanced Analysis

## Architectural Role

This file serves as the **server-side client lifecycle manager** within the Game VM subsystem (part of `code/game`). It bridges the authoritative server (`code/server/sv_game.c`) into game logic by implementing the four cardinal VM entry points for client state transitions: `ClientConnect`, `ClientBegin`, `ClientSpawn`, `ClientDisconnect`. These functions are dispatched from the server's `SV_GameSystemCalls` dispatcher when the engine invokes `VM_Call(gvm, GAME_CLIENT_*)`. The file is also home to the spatial spawn-point selection hierarchy—a core component of the level geometry → player-placement pipeline that the renderer and collision system depend on indirectly through PVS-culled entity snapshots.

## Key Cross-References

### Incoming (who depends on this file)
- **Server syscall dispatch** (`code/server/sv_game.c`): Calls `ClientConnect`, `ClientBegin`, `ClientSpawn`, `ClientDisconnect` via the game VM's `vmMain` entry and `SV_GameSystemCalls` dispatcher.
- **Game module lifecycle** (`code/game/g_main.c`): Calls `InitBodyQue` at map load time; also provides `CalculateRanks` to update scoreboard state triggered by client lifecycle events.
- **Respawn pipeline** (`code/game/g_combat.c`): Calls `CopyToBodyQue` when a player dies, before `ClientSpawn` is called for respawn.
- **Bot integration** (`code/game/g_bot.c`): Called for bot-specific spawning; bot clients flow through the same `ClientConnect` → `ClientBegin` → `ClientSpawn` path.

### Outgoing (what this file depends on)
- **Per-frame client logic** (`code/game/g_active.c`): Calls `ClientThink` and `ClientEndFrame` to settle player state after spawn; both are defined elsewhere.
- **Team logic** (`code/game/g_team.c`): Calls `SelectCTFSpawnPoint` for team-based spawns; `SelectSpawnPoint` is the fallback for deathmatch-only spawns.
- **Bot AI** (`code/game/g_bot.c`, `code/game/ai_main.c`): Calls `G_BotConnect`, `BotAIShutdownClient` for lifecycle; also consumes via `trap_BotLib*` syscalls at spawn time.
- **Shared background code** (`code/game/bg_misc.c`, `code/game/bg_pmove.c`): `BG_PlayerStateToEntityState` bridges authoritative `playerState_t` to network-transmitted `entityState_t` after spawn.
- **Engine services** (via `trap_*`): Collision (`trap_Trace`, `trap_PointContents`), entity linking (`trap_LinkEntity`, `trap_UnlinkEntity`), networking (`trap_SetConfigstring`), sound/effects (`trap_BotLib*`), debug (`G_LogPrintf`).

## Design Patterns & Rationale

**Object pooling (body queue):** Pre-allocates `BODY_QUEUE_SIZE=8` corpse entities at init to avoid runtime `malloc` calls within `CopyToBodyQue`. This is idiomatic for late-1990s game engine design when frame-rate consistency was paramount; LIFO circular allocation (`bodyQueIndex = (bodyQueIndex + 1) % BODY_QUEUE_SIZE`) ensures bodies are recycled fairly.

**Hierarchical spawn selection:** `SelectSpawnPoint` → `SelectRandomFurthestSpawnPoint` → list insertion sort by distance creates a ranked spawn list biased toward the top half (via `rnd = random() * (numSpots / 2)`). This heuristic discourages players from spawning too close to their death point while avoiding starvation of distant spots.

**State machine bootstrap:** The four lifecycle functions (`ClientConnect`, `ClientBegin`, `ClientSpawn`, `ClientDisconnect`) form a **linear state machine** from the game module's perspective: `CONNECT` → `BEGIN` → (respawn loops through `SPAWN`) → `DISCONNECT`. The server enforces the legality of transitions; the game module assumes proper sequencing.

**Deferred entity think chains:** Bodies and other entities use `think` function pointers and `nextthink` timestamps (set by `BodySink` at +5000ms) rather than hardcoding simulation loops. This allows the engine to defer work and maintain a single frame loop in `g_main.c`.

## Data Flow Through This File

1. **Client joins:** Server invokes `ClientConnect(clientNum)` → validates IP ban + password, reads session data, calls `ClientUserinfoChanged` to broadcast userinfo configstring.
2. **Client ready:** Server invokes `ClientBegin(clientNum)` → initializes gentity_t, calls `ClientSpawn`.
3. **Spawn/respawn:** `ClientSpawn(ent)` selects spawn point (via `SelectInitialSpawnPoint` or `SelectSpawnPoint` depending on `g_gametype`), resets most of `gclient_t` while preserving persistent fields, calls `ClientThink` + `ClientEndFrame` to seed initial frame state, then fires spawn-point trigger targets via `G_UseTargets`.
4. **Death:** Combat code calls `CopyToBodyQue(ent)` → copies entity state into pooled body, schedules `BodySink` think at +5000ms.
5. **Disconnect:** Server invokes `ClientDisconnect(clientNum)` → tosses items, tears down followers, clears entity, recalculates ranks.

## Learning Notes

**Idiomatic to this era:** The commented-out dead code block in `SelectSpawnPoint` (alternating nearest-avoidance retry logic) suggests the final `SelectRandomFurthestSpawnPoint` algorithm was a later refinement. This is typical of shipped 1999–2005 codebases where optimization passes left behind pedagogically useful "what we tried first" artifacts.

**Modern engine divergence:** Contemporary engines (Unreal, Unity) use **type-erasure** via `Pawn`/`Actor` base classes or component systems to unify spawn logic; Quake III's dual-VM architecture (server `game`, client `cgame`) necessitates careful `playerState_t` marshaling in `BG_PlayerStateToEntityState` to keep prediction in sync.

**Spawn-point semantics:** The `initial`, `nobots`, `nohumans` flags show how entity properties encode gamemode-specific constraints. Modern engines would use tags or explicit spawn-pool registration; this approach minimizes metadata overhead in the `.map` file format.

## Potential Issues

- **Body queue exhaustion:** If more than 8 corpses exist simultaneously, the 9th respawn will silently overwrite a body that may still have `nextthink > level.time` (scheduled but not yet sunk). Under normal gameplay this is fine, but custom `BODY_QUEUE_SIZE` edits could hide subtle race conditions.
- **Spawn selection O(n²) insertion sort:** `SelectRandomFurthestSpawnPoint` performs O(n²) distance comparisons. On maps with >64 spawn points, the list truncates silently (clamping to 64), potentially starving far points. A heap-based approach or incremental sort would be more robust.
- **Telefrag check races:** `SpotWouldTelefrag` uses a box sweep, but between the check and `trap_LinkEntity` in `ClientSpawn`, another client could spawn at the same location. The `G_KillBox` call in `ClientSpawn` mitigates this post-hoc, but there's a narrow window.
