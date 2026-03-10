# code/game/g_rankings.c — Enhanced Analysis

## Architectural Role

This file implements the **server-side telemetry collection layer** for Quake III's global online rankings system. It bridges game-logic events (weapon fires, damage, deaths, item pickups) and session metadata to an external ranking service via trap-call syscalls. Acts as a stat aggregator and rules enforcer for ranked matches—maintaining client auth states, enforcing competitive rules (bot bans, limit caps), and flushing per-player match ratings at game-over.

## Key Cross-References

### Incoming (who depends on this file)
- **code/game/g_main.c** → calls `G_RankRunFrame()` every frame from the main game loop (`G_RunFrame`)
- **code/game/g_combat.c** → calls `G_RankDamage()`, `G_RankPlayerDie()` on hit/death events
- **code/game/g_weapon.c** → calls `G_RankFireWeapon()` on weapon discharge
- **code/game/g_items.c** → calls `G_RankPickup*()` family on item pickup events
- **code/game/g_client.c** → calls `G_RankClientDisconnect()` on disconnect/session end
- **code/game/ai_main.c** → indirectly; bots are kicked if rankings become active

### Outgoing (what this file depends on)
- **Engine layer (trap_Rank*)** → `trap_RankCheckInit`, `trap_RankBegin`, `trap_RankPoll`, `trap_RankActive`, `trap_RankUserStatus`, `trap_RankReportInt`, `trap_RankReportStr`, `trap_RankUserReset` (all syscalls to `code/server/sv_rankings.c`)
- **Player management** → `ClientSpawn()`, `SetTeam()`, `OnSameTeam()` for enforcing ranked-match spectator/active transitions
- **Client communication** → `trap_SendServerCommand()`, `trap_SendConsoleCommand()` for status/menu updates and bot kicks
- **Cvar layer** → `trap_Cvar_Set()`, `trap_Cvar_VariableStringBuffer()`, `trap_Cvar_VariableIntegerValue()` for match limits and config metadata
- **Scoreboard** → `DeathmatchScoreboardMessage()` to refresh ranks immediately when players activate

## Design Patterns & Rationale

**Event-driven async telemetry:** Rather than the game VM exporting raw stats files, each combat/item event triggers an immediate trap call, allowing the engine to batch/queue reports to a remote service without blocking the game loop.

**Frame-aware deduplication:** Static variables in `G_RankDamage()` (last_framenum, last_self, last_attacker, last_means_of_death) track the previous hit context. Shotgun fires multiple pellets per frame; only the first is counted as a "new hit" to avoid inflating hit statistics. This pattern—persisting state across calls within a module—is typical for Q3's event aggregation where the frame boundary is the natural atomicity unit.

**Warmup gating:** All reporting functions return early if `level.warmupTime != 0`, ensuring practice rounds don't pollute stats. This is enforced at the trap level, not the game logic level.

**Ranked-game rule enforcement:** `G_RankRunFrame()` uses the rankings service's per-client status to decide whether bots are allowed, whether the client is eligible to play, and what team they may join. The status enum (QGR_STATUS_ACTIVE, QGR_STATUS_NEW, QGR_STATUS_NO_USER, etc.) acts as a state machine.

## Data Flow Through This File

1. **Initialization** (`G_RankRunFrame`): Engine asks rankings service if it's initialized; if not, calls `trap_RankBegin(GR_GAMEKEY)` once. Each frame polls (`trap_RankPoll`) and checks if service is active.

2. **Client status tracking**: Each frame iterates all clients, fetches their auth status from the service. If status changes, sends `rank_status` command to client and adjusts team/spectator state. On activation, notifies all active clients that they played together.

3. **Event reporting**: Combat/pickup events call `trap_RankReportInt()` with (self, opponent/-1, QGR_KEY_*, value, flag). For damage, reports general+specific hit/damage/splash keys; for deaths, reports general+weapon-specific key; for items, reports item type.

4. **Session end**: `G_RankGameOver()` calls `G_RankClientDisconnect()` for each active client (computing their match rating), then reports session metadata (hostname, map, gametype, limits, server config, version) as string and integer pairs.

## Learning Notes

**Q3-era competitive infrastructure:** The rankings system reveals Q3A's design assumption that an operator-run centralized ranking service would exist (now defunct). The stat vocabulary (QGR_KEY_*) is weapon/item-aware, enabling per-weapon win rates and per-item usage patterns to feed into player skill ratings.

**Trap boundary discipline:** All outbound communication uses trap calls; no direct file I/O or network calls. This maintains the VM sandbox: the game logic is stateless with respect to persistence—the engine/server owns the ranking service channel.

**Deferred team assignment:** Players connecting to a ranked server start as spectators until their auth status becomes ACTIVE, at which point `SetTeam(..., "free")` moves them to a joinable team. This prevents unranked players from affecting match stats.

**Static state idiom:** File-static variables in `G_RankDamage()` are not thread-unsafe in Q3A because the game VM is single-threaded and frame-serialized by the engine. This is idiomatic for the era and engine design.

Modern engines would use per-client stat aggregators (e.g., ECS-style stat component) or a dedicated telemetry subsystem, whereas Q3A's functional dispatch at event sites is characteristic of its event-callback architecture.

## Potential Issues

- **Dead code block** (line ~77): `if (i == 0) { int j = 0; }` in `G_RankRunFrame()` is vestigial debug code left behind.
- **Lack of error handling**: If `trap_RankReportInt()` or `trap_RankReportStr()` fail silently, the game has no way to detect or recover. Failures are invisible to game logic.
- **Grenade entity check** (line ~382): Comment notes attacker index is a grenade-shooter entity (index 245), not a client. Guard `if (g_entities[attacker].client)` prevents crashes but silently loses stats for grenade-splash givers. Similar issue in friendly-fire block.
