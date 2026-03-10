Looking at the cross-references and architecture context, I can now write a thorough second-pass analysis that adds cross-cutting insights.

# code/game/ai_dmq3.h — Enhanced Analysis

## Architectural Role

This header serves as the **public interface between the per-bot FSM layer** (`ai_dmnet.c`) **and the deathmatch-specific decision-making logic** (`ai_dmq3.c`). It also bridges the game VM's bot system to the lower-level botlib AAS navigation library. Functions declared here are driven by the server's main frame loop (via `SV_BotFrame` → `ai_main.c` → `BotDeathmatchAI`) and called by the FSM (`AINode_*` functions in `ai_dmnet.c`) to make tactical decisions. The file essentially encapsulates all the heuristics that determine bot behavior in deathmatch and CTF modes—enemy detection, weapon/item selection, goal routing, and aggression balancing.

## Key Cross-References

### Incoming (who depends on this file)
- **`ai_main.c`** — Calls `BotSetupDeathmatchAI`, `BotShutdownDeathmatchAI`, and invokes `BotDeathmatchAI` once per frame per bot from the server loop
- **`ai_dmnet.c`** — The per-bot FSM state machine (`AINode_*` functions) calls decision functions like `BotFindEnemy`, `BotWantsToRetreat`, `BotWantsToChase`, `BotCTFSeekGoals` to determine next state
- **`ai_team.c`** — Queries team-play predicates and calls goal-setting functions during team objective coordination
- **`ai_chat.c`** — May reference bot state fields and predicates for generating contextual chat messages
- **`g_bot.c`** — Initializes bot clients and wires them into the game entity system

### Outgoing (what this file depends on)
- **`botlib` (via `trap_BotLib*` syscalls)** — All pathfinding, movement simulation, and entity queries (`AAS_PointAreaNum`, `AAS_TraceClientBBox`, etc.) flow through syscall range 200–599
- **`ai_main.h`** — Defines `bot_state_t`, the central mutable context passed to nearly every function declared here
- **`be_aas.h` / botlib AAS layer** — Provides `aas_entityinfo_t` for entity queries in predicates like `EntityIsDead`, `EntityIsInvisible`
- **Game globals** — Reads `gametype`, `maxclients` to branch behavior (CTF vs. FFA vs. Team Arena modes)
- **Bot cvars** — Reads `bot_grapple`, `bot_rocketjump`, `bot_fastchat`, etc. to modulate bot capabilities and style

## Design Patterns & Rationale

### Predicate Queries
Functions like `BotIsDead`, `BotIsObserver`, `EntityIsInvisible` are **lightweight state predicates** — typically one or two field accesses on `bot_state_t` or `aas_entityinfo_t`. These are called frequently in decision loops and deliberately kept simple to avoid redundant computation.

### Two-Level Decision Hierarchy
- **Low-level**: Simple predicates (`BotIsDead`, `BotInLavaOrSlime`)
- **High-level**: Complex heuristics (`BotAggression`, `BotFeelingBad`, `BotWantsToRetreat`) that aggregate multiple predicates and weighted scoring. This mirrors the FSM pattern where state nodes query decision functions to determine transitions.

### CTF / Mission Pack Branching
Rather than runtime dispatch tables, `#ifdef MISSIONPACK` gates entire function groups (`Bot1FCTFSeekGoals`, `BotHarvesterSeekGoals`). This is a compile-time strategy that reduces runtime overhead but couples game modes tightly to the codebase. Modern engines would use a runtime mode enum and dispatch table instead.

### Extern Cvars for Tuning
Bot behavior is parameterized by `vmCvar_t` globals (`bot_grapple`, `bot_rocketjump`, etc.) rather than hardcoded constants. This allows designers/server admins to tune bot difficulty and style via `bot_*.cfg` files without recompilation—idiomatic for Q3's design era.

### Waypoint System
`BotCreateWayPoint`, `BotFindWayPoint`, `BotFreeWaypoints` manage hand-authored waypoint graphs for navigation. This predates modern navmeshes and node graphs; waypoints are simple named spatial landmarks that bots can path between via the AAS system.

## Data Flow Through This File

**Inputs:**
1. Per-frame server state encapsulated in `bot_state_t` (position, health, inventory, current goal, enemy)
2. Entity visibility and state queried from the AAS entity layer
3. Game mode (`gametype`) and configuration (`bot_*` cvars)

**Core Processing:**
1. **Enemy Detection** (`BotFindEnemy`) — Scans for visible enemies, considers team membership, updates `bs->enemy`
2. **Situational Assessment** (`BotAggression`, `BotFeelingBad`) — Evaluates threat level, health, item availability to decide tactical posture
3. **Goal Selection** (`BotCTFSeekGoals`, `BotWantsToRetreat`) — Routes the bot toward objectives (flags in CTF, roaming in FFA)
4. **Weapon/Item Optimization** (`BotChooseWeapon`, `BotUpdateInventory`) — Selects best weapon and consumables for current situation
5. **Movement Execution** (`BotAttackMove`, `BotAimAtEnemy`) — Translates decisions into movement commands and aiming

**Outputs:**
1. Updated `bot_state_t` fields (current enemy, goal, weapon, movement direction)
2. Movement/aiming commands synthesized via EA (elementary action) layer of botlib, converted to `usercmd_t`
3. Implicit: Pathfinding requests to AAS layer via syscalls, which populate the navigation stack in `bot_state_t`

## Learning Notes

### Idiomatic Q3 Bot AI Patterns
- **Mutable-context procedural style**: Unlike ECS or component-based engines, the bot AI is fundamentally procedural—a series of function calls that mutate a central `bot_state_t` structure. There's no "push message to event queue" or "update component"; just direct field assignment.
- **Extern global state**: Game-wide variables like `gametype` and cvars are accessed directly, not injected. This is fast and simple but couples the code tightly to the global namespace.
- **Simple heuristics over learning**: Aggression, fear, and retreat logic are hand-tuned fuzzy scores, not learned. Appropriate for a game released in 1999; modern bots would use behavior trees or planners.
- **Waypoint-centric navigation**: Unlike modern navmeshes covering entire levels, Q3 bots use hand-placed waypoint collections managed at runtime. More flexible than precomputed coverage but requires manual level-by-level tuning.

### Differences from Modern Engine Patterns
- No dependency injection; no vtables for game-mode variants
- No behavior tree or planner abstraction; FSM logic is sprawled across `ai_dmnet.c` state nodes
- No "desired state" message passing; decisions directly mutate shared `bot_state_t`
- Conditional compilation (`#ifdef`) for game variants instead of runtime polymorphism

### Connection to Engine Concepts
- **AAS as a navigation substrate**: The entire bot movement and sensing pipeline depends on the AAS (Area Awareness System)—a space-partitioning abstraction specific to Quake engines. Unlike voxels or navmeshes, AAS computes reachability links on-the-fly and allows bots to exploit complex geometry (jumppads, teleporters, ladders).
- **Syscall boundary**: All botlib calls (pathfinding, entity queries, movement simulation) cross the VM boundary via syscalls. This isolation is intentional but expensive—each decision may trigger multiple round-trips.

## Potential Issues

- **No visibility or locking on extern globals**: `gametype` and `maxclients` are read directly; if the server were multithreaded, these would require atomic access.
- **Cvar reads without caching**: Functions may read volatile cvars multiple times per frame (e.g., checking `bot_grapple` in multiple decision functions). Modern code would cache and validate at frame boundaries.
- **FSM-side effects**: Functions return void and mutate `bs` in-place. Hard to test in isolation; no clear pre/post conditions.
- **Waypoint hardcoding per map**: Each map's waypoint placement is baked at compile time. Errors in waypoint topology can strand bots indefinitely.
- **CTF-specific code duplication**: `BotCTFSeekGoals` vs. `Bot1FCTFSeekGoals` duplicates logic; modern code would parameterize the difference.
