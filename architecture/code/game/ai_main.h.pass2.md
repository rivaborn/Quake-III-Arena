# code/game/ai_main.h — Enhanced Analysis

## Architectural Role
This header anchors the **Game VM's in-game bot AI subsystem**—a procedural per-bot FSM layer sitting atop botlib's self-contained navigation and decision library. The monolithic `bot_state_t` structure serializes all per-bot runtime state (movement, combat, team goals, formation, time-based behavior gates), while the `ainode` function-pointer dispatch enables frame-by-frame state transitions. `ai_main.h` acts as the **glue between botlib** (which handles pathfinding, reachability, weapon/goal scoring in isolation) **and the game frame loop** (which evolves bot state, resolves combat, and synchronizes team data each frame via trap syscalls).

## Key Cross-References

### Incoming (who depends on this file)
- **All game AI FSM modules**: `ai_dmq3.c` (deathmatch FSM), `ai_dmnet.c` (team/CTF FSM), `ai_team.c` (team tactics), `ai_chat.c` (team communication) — all include this header and operate on `bot_state_t`
- **Bot lifecycle**: `g_bot.c` (`G_BotConnect`, `G_BotUserinfoChanged`, `G_BotThink`) creates, configures, and ticks bots each frame
- **Server frame loop**: `sv_bot.c` calls `BotAI` entry points each frame via syscall range 200–599

### Outgoing (what this file depends on)
- **botlib library** (`botlib.h`, `be_aas.h`): All navigation data (`bot_goal_t`), settings (`bot_settings_t`), entity queries (`aas_entityinfo_t`), and traces (`bsp_trace_t`) are sourced from botlib; game AI never reimplements pathfinding
- **Game module bridge functions**: `BotAI_Trace`, `BotAI_Print`, `BotAI_GetClientState`, `BotAI_GetEntityState`, `BotAI_GetSnapshotEntity`, `BotTeamLeader` wrap trap syscalls to botlib engine imports
- **Shared game/client code**: `bg_public.h` (player/entity state), `q_shared.h` (types and math)
- **Global game state**: `floattime` (extern from `common.c`), entity/client queries via syscalls

## Design Patterns & Rationale

| Pattern | Rationale |
|---------|-----------|
| **Function-pointer FSM dispatch** (`ainode`) | Enables lightweight state-machine transitions without a switch statement; supports dynamic behavior swapping (e.g., chase → strafe → retreat) |
| **Monolithic state aggregation** (`bot_state_t`) | Simplifies frame-by-frame serialization and debugging; all bot data lives in one malloc'd block; trade-off: ~5000 bytes per bot (fragile for large maps) |
| **Dual-layer AI architecture** (botlib + game FSM) | Separates reusable navigation/scoring library (botlib, VMS-agnostic) from game-specific FSM logic; enables DLL/QVM swapping of game code without recompiling botlib |
| **Time-accumulation behavior gating** (100+ `*_time` fields) | Avoids callback/event overhead; simple comparison `if (ltime - lastchat_time > 10.0f) { chat() }` gates frame-by-frame decisions; brittle on framerate changes |
| **Activation goal stack** (`bot_activategoal_t`, max 8 entries) | Handles nested trigger interactions (e.g., button A opens door B which blocks area C); allows bot to queue button activations while traversing |
| **Formation/relative positioning** (`formation_*` fields) | Enables coordinated team movement; bot aims to maintain angle/distance relative to a lead teammate rather than absolute waypoint |

## Data Flow Through This File

**Frame-by-frame bot AI loop:**
1. **Input phase** (from game state):
   - Server snapshot → `cur_ps`, `origin`, `velocity`, `areanum` (AAS point-to-area lookup)
   - Enemy visibility from trace queries (`BotAI_Trace`)
   - Inventory counts from entity state snapshots
   - Flag/objective state from configstring queries
   
2. **Processing phase** (FSM + botlib):
   - Call `ainode(bs)` — routes to combat/roaming/CTF logic based on `ltgtype`
   - Botlib goal selection: fuzzy-score items/enemies, pick best `bot_goal_t`
   - Botlib movement FSM: execute travel type (walk/jump/swim/ladder) to reach goal
   - Update time accumulators: `ltime += frametime`; gate weapon changes, chat, item checks
   
3. **Output phase**:
   - Botlib EA layer synthesizes `usercmd_t` from predicted movement
   - Game code transmits command to network (delta-compressed, rate-limited)
   
4. **Feedback phase**:
   - Damage inflicted → update `lasthealth`, trigger death logic
   - Kill/death → update `num_kills`, `revenge_enemy`, scoring
   - Teammate state changes → update team goal (`ltgtype`, `teamgoal`)

**Team/CTF state synchronization:**
- Flag status (`redflagstatus`, `blueflagstatus`, `neutralflagstatus`) updated from game via `G_BotUpdateTeamGoal`
- Formation state (`formation_dir`, `formation_origin`) rebuilt each frame if bot is in formation
- Shared objectives (`lead_teammate`, `teamgoal`) broadcast to squad via text chat

## Learning Notes

- **Procedural AI era (mid-2000s):** No behavior trees, no data-driven scripts — all logic hardcoded in FSM nodes. A modern engine would use ECS (entity components) or hierarchical behavior trees.
- **Monolithic vs. componentized:** `bot_state_t` would decompose into ~10 components in modern design (movement state, combat state, team state, etc.), enabling hot-reload and composition.
- **Navigation layer divorce:** Botlib is **completely decoupled** from game logic; this enabled id to license botlib separately and reuse it in multiple games (Q3, RTCW, ET).
- **Waypoint-based patrolling:** Predates modern navmeshes; checkpoint/patrol routes are manually authored linked lists, not procedurally generated.
- **CTF strategy constants** (`CTFS_AGRESSIVE`) show hardcoded mid-2000s competitive tweaks; modern AI would learn strategy from replays or player input.
- **Chat system:** Template-matching string replacement (not generative); `be_ai_chat.c` selects canned responses.
- **No multi-threading:** All bot AI runs on the main server frame; no async pathfinding pre-computation.
- **Timescale fragility:** 100+ time fields assume fixed 125 Hz server tick; variable framerate breaks behavior timing.

## Potential Issues

- **Memory overhead:** ~5000 bytes × 64 max bots = 320 KB just for per-bot state; no pooling or lazy allocation.
- **Global `floattime` thread-safety:** Shared float accessed by bot AI, renderer, and physics without synchronization; could cause divergence in SMP mode.
- **Activation stack overflow:** `MAX_ACTIVATESTACK = 8`; complex nested triggers (e.g., multi-button puzzle) could silently fail if stack fills.
- **Duplicate goal storage:** `lastgoal_teamgoal` mirrors `teamgoal` without documented sync points; risk of state drift after order changes.
- **Hardcoded constants:** `MAX_ITEMS = 256`, `MAX_PROXMINES = 64` baked into struct; changing requires recompile and network compatibility breaks.
- **No introspection helpers:** No getter/setter functions for time comparisons; direct field access scattered across all `ai_*.c` files makes refactoring risky.
