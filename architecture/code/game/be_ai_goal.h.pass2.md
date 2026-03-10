# code/game/be_ai_goal.h — Enhanced Analysis

## Architectural Role

This header declares the **goal selection subsystem** of botlib, the decision-making layer responsible for answering "what should this bot pursue?" It sits at the boundary between botlib (self-contained AI library) and the game VM (server-side game logic). Goals represent *targets* (items, camp spots, map locations) that the pathfinding system (`be_aas_route.c`) then computes routes to. The goal system is stateful per-bot (via opaque `goalstate` handles) and driven by fuzzy-logic item weights, allowing evolving bot personalities.

## Key Cross-References

### Incoming (who depends on this file)
- **Game VM** (`code/game/ai_dmnet.c`, `ai_dmq3.c`) — FSM state nodes call `BotChooseLTGItem`/`BotChooseNBGItem`/goal stack ops via **trap_BotLib\*** syscalls routed through the server
- **Server** (`code/server/sv_bot.c`) — Bridges all syscalls from game VM to botlib functions; calls `BotSetupGoalAI`, `BotAllocGoalState`, etc. directly
- **Bot lifecycle** (`code/game/g_bot.c`) — Allocates/frees goal state per bot; loads/unloads item weights per bot
- **Fuzzy logic genetic algorithm** — Offline tools mutate/interbreed goal logic for bot evolution

### Outgoing (what this file depends on)
- **AAS navigation** (`be_aas.h`, `be_aas_route.c`) — Uses area numbers, travel flags, reachability queries to score goals by distance/accessibility
- **Entity system** (`be_aas_entity.c`, `be_aas_sample.c`) — Queries entity positions and spatial sampling for dynamic items (dropped weapons, CTF flags)
- **Item database** — Internal level-item registry built at map load; dynamic list updated per frame
- **Shared types** (`q_shared.h`) — `vec3_t` for positions and bounding boxes
- **Implementation** (`code/botlib/be_ai_goal.c`) — All function bodies defined there; only declarations exposed here

## Design Patterns & Rationale

**Opaque Handle + Service Table Pattern**  
Goal state is identified by an integer handle, not a pointer. This allows botlib to hide the state table behind the `botlib_export_t` vtable, preventing direct linking and enforcing strict sys-call boundaries.

**LTG (Long-Term Goal) vs. NBG (Nearby Goal) Dichotomy**  
Strategic planning is two-tier: pick a distant objective (LTG) via fuzzy logic over all items, then look for unpenalizing detours nearby (NBG). This mimics human navigation (go to base, but grab health if it's on the way).

**LIFO Goal Stack (depth=8)**  
Enables hierarchical task execution: push a high-priority interrupt goal (flee to armor) on top of the current goal stack; when popped, resume. Depth limit suggests simplified planning relative to modern BT/HTN systems.

**Temporal Avoid List (cap=256)**  
Rather than instantly forgetting visited goals, a per-bot blacklist with configurable decay timers prevents looping. Entries are explicitly set via `BotSetAvoidGoalTime`, giving the AI FSM control over "why" a goal was avoided.

**Fuzzy Logic for Personality Variation**  
Item weights are externally loaded from `.c` script files, not hard-coded. Enables:
- Bot personality tuning (aggressive = favor weapons; defensive = favor armor)
- Genetic algorithm evolution in offline tools
- Deterministic bot differentiation without code duplication

**Separation of **Selection** from **Execution**  
Goal selection (this module) is decoupled from pathfinding (AAS) and movement execution (`be_ai_move.c`). Each layer is independently testable and swappable.

## Data Flow Through This File

**Map Initialization → Per-Bot Spawn → Per-Frame Loop → Bot Disconnect**

```
BotSetupGoalAI()
  ↓ (global init)
BotInitLevelItems() + BotUpdateEntityItems() [periodic]
  ↓ (static + dynamic item registries)
BotAllocGoalState(client) per bot
  ↓
BotLoadItemWeights(goalstate, "configs/bots/*.c")
  ↓ (fuzzy logic weights loaded)
[Each Frame]
  BotChooseLTGItem(goalstate, origin, inventory, travelflags)
    → Scored against all level items; returns best match
  ↓
  BotChooseNBGItem(goalstate, ..., ltg, maxtime)
    → Scored only items reachable without major detour from LTG
  ↓
  BotPushGoal(goalstate, chosen_goal)
    → Enqueued in per-bot LIFO stack (depth≤8)
  ↓
  [Movement FSM (`be_ai_move.c`) pops + executes]
  ↓
  BotAvoidGoalTime() / BotSetAvoidGoalTime()
    → Decay cooldowns; mark goals as "avoid for N seconds"
BotFreeItemWeights(goalstate) + BotFreeGoalState(handle) on disconnect
  ↓
BotShutdownGoalAI() at botlib unload
```

**Cross-Subsystem Data**: Goals reference AAS area numbers, entity IDs, and world positions; scoring uses AAS travel-time estimates and inventory state.

## Learning Notes

1. **Fuzzy Logic as Extensibility Point** — Rather than hard-code "bot likes plasma rifles," weights are loaded from external files. Early example of data-driven behavior.
2. **Handle-Based Lifecycle** — Integer handles hide state allocation; enables reference counting and resource pooling across the sys-call boundary.
3. **Stateless Interface** — Each function is pure given its inputs; no global mutable goal state exposed. Allows frame-coherent multi-bot updates.
4. **Temporal Dynamics** — Avoid list with decay timers is a simple form of short-term memory; prevents thrashing without requiring sophisticated learning.
5. **Era-Specific Simplicity** — Stack depth=8, avoid cap=256 reflect late-1990s constraints. Modern engines use HTNs, behavior trees, or blackboard systems for richer hierarchies.
6. **Boundary Between Logic & Navigation** — This module answers *what*; AAS answers *how*. Clear responsibility partition.

## Potential Issues

- **No visible validation** in the header of goal handles or goal validity. Bounds checking deferred to implementation.
- **Static goal stack size** (8) could bottleneck complex hierarchical planning; no dynamic allocation fallback mentioned.
- **Avoid goal capacity** (256) is a hard ceiling; no mechanism shown to handle capacity exhaustion gracefully.
- **Item weight files** are loaded externally; no visible error recovery if files are missing or corrupt (likely handled in implementation but opaque here).
- **No versioning** in the function signatures; evolution would require adding new syscall opcodes rather than extending existing ones.
