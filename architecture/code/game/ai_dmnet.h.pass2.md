# code/game/ai_dmnet.h — Enhanced Analysis

## Architectural Role

This header file declares the **finite-state machine (FSM) backbone** of Quake III's deathmatch bot AI. It sits at the boundary between `ai_main.c` (which drives bot logic each frame by calling the current state function) and `ai_dmnet.c` (which implements those state behaviors using botlib services). The FSM abstracts bot behavior into discrete, named states—ranging from lifecycle states (`Intermission`, `Respawn`) through tactical layers (`Seek_LTG`, `Battle_Fight`)—allowing bots to compose complex decision trees while keeping each state's logic isolated and testable.

## Key Cross-References

### Incoming (who depends on this file)

- **`ai_main.c`**: Each frame, calls `bs->ainode(bs)` (a function pointer initialized by `AIEnter_*` calls). This is the primary driver of bot behavior.
- **`ai_dmq3.c`**: Deathmatch-specific AI logic likely calls `AIEnter_*` to transition states during decision-making (e.g., when spotting an enemy, transitioning from `Seek_LTG` to `Battle_Fight`).
- **Team AI files** (`ai_team.c`, etc.): May use the same FSM backbone or call entry functions to coordinate team-wide state changes.
- **`ai_dmnet.c`**: Implements all declared `AIEnter_*` and `AINode_*` functions; the implementations read `bot_state_t` fields and issue bot commands through botlib.

### Outgoing (what this file depends on)

- **`ai_main.h` / `g_local.h`**: Defines `bot_state_t`, including the `bs->ainode` function pointer, `bs->cur_ps` (playerstate), `bs->origin`, `bs->enemy`, and other state fields.
- **`ai_dmnet.c`**: Provides all function implementations; also calls botlib (`trap_BotLib*` syscalls) for pathfinding, movement simulation, goal selection, reachability checks.
- **Botlib** (via `trap_BotLib*` syscalls): Used by state implementations for AAS queries, route caching, movement prediction, and entity tracking.

## Design Patterns & Rationale

**State/Strategy Pattern**: Each `AINode_*` function encapsulates one behavioral state and returns control to the caller; `AIEnter_*` functions switch the bot's active strategy by updating `bs->ainode`. This avoids massive switch/case statements and allows easy addition of new bot behaviors without modifying existing code.

**Circular History Buffer**: `MAX_NODESWITCHES = 50` and the `BotDumpNodeSwitches()` diagnostic hook implement a classic bounded history for debugging pathological AI loops. If a bot switches states 50+ times in a single frame, `BotRecordNodeSwitch` (internal to `ai_dmnet.c`) logs it, preventing infinite loops from silently corrupting gameplay.

**Function Pointers Over Inheritance**: Pre-C++11 code avoids vtable overhead; the `bs->ainode` pointer is a lightweight way to select behavior at runtime without VM/language features.

---

## Data Flow Through This File

**Input**: Each server frame, `ai_main.c` holds a `bot_state_t` with the bot's current position, enemy reference, inventory, health, and `ainode` function pointer.

**Processing**:
1. `ai_main.c` calls `bs->ainode(bs)` → executes the current state's logic (defined in `ai_dmnet.c`).
2. The state function reads bot context (position, enemy, inventory) and may:
   - Continue in the current state (return non-zero).
   - Call `AIEnter_<NextState>()` to transition, which records the switch and updates `bs->ainode`.
3. Botlib queries (via `trap_BotLib*`) inform decisions: "Can I reach that goal?" "Where is the nearest health?"

**Output**: The state function issues a `bot_input_t` (forward/strafe/jump/fire commands) via the botlib EA layer, or triggers a state transition.

**Stateful Transitions**: Over a bot's lifetime, the FSM path might look:  
`Intermission` → `Observer` → `Respawn` → `Stand` → `Seek_LTG` → (enemy spotted) → `Battle_Chase` → `Battle_Retreat` → (health low, regroup) → `Seek_NBG` (nearby goal) → …

---

## Learning Notes

- **Classic FSM with stateless transitions**: Unlike ECS or behavior trees, each state is independent; there is no global "state stack" or hierarchical context. This mirrors early game engines and remains performant.
- **Idiomatic debugging practice**: The `node_switch_history` and `BotDumpNodeSwitches()` pattern reflects the era's tooling constraints—no integrated debuggers for runtime game logic, so AI developers instrumented FSM logs.
- **Behavioral modularity**: The 11+ states (`Intermission`, `Observer`, `Respawn`, `Stand`, `Seek_ActivateEntity`, `Seek_NBG`, `Seek_LTG`, `Seek_Camp`, `Battle_Fight`, `Battle_Chase`, `Battle_Retreat`, `Battle_NBG`) reflect a clean separation of concerns: lifecycle, goal-seeking, and combat tactics are independent FSM layers.
- **Modern alternative**: Contemporary engines use behavior trees, utility AI scorers, or planners to achieve similar modularity with more declarative semantics. The Q3 FSM is lower-level but more transparent.

---

## Potential Issues

1. **Dead Code**: `AIEnter_Seek_Camp` is declared but no `AINode_Seek_Camp` is declared. Either `Seek_Camp` transitions immediately to another state, or the node implementation was never completed. Check `ai_dmnet.c` to verify.

2. **Fixed-Size History Limit**: `MAX_NODESWITCHES = 50` may not catch all infinite-loop bugs if multiple bots each switch 50 times in one frame. A more robust approach would be per-bot switch counters or frame-time budgets.

3. **No State Validation**: There's no enforcement that `bs->ainode` points to a valid function. A corrupted pointer could cause a crash. (This is typical for the era; modern code would use enums or tagged unions.)

4. **Stateless Nodes**: If two bots need coordinated behavior (e.g., team tactics), they must coordinate entirely through game-world state (entity positions, shared goals). The FSM itself offers no context-passing or parent-state semantics.

---
