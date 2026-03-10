# code/game/ai_dmnet.c — Enhanced Analysis

## Architectural Role

This file implements the bot AI **finite-state machine orchestrator** at the top level of the game VM's decision-making stack. It is NOT part of botlib; rather, it sits *above* botlib and decides **what** goals the bot should pursue based on game state, team objectives, and player interactions. Botlib (accessed via `trap_BotLib*` syscalls) then handles **how** to navigate to those goals. The file bridges three major layers: the server's game logic (`ai_main.c`, `g_*.c`), the botlib navigation library, and the engine's core services (traces, entity state, chat).

## Key Cross-References

### Incoming (who depends on this)
- **`ai_main.c`**: Calls `BotResetNodeSwitches` at frame start and invokes `bs->ainode()` (function pointer to an `AINode_*`) each frame
- **`ai_dmq3.c`/`ai_team.c`**: Call `BotGetLongTermGoal` and other goal-selection helpers to coordinate multi-bot behavior
- **`g_bot.c`**: Initializes bot state and links to FSM entry point

### Outgoing (what this calls)
- **botlib via `trap_BotLib*` syscalls**: `trap_BotChooseLTGItem`, `trap_BotChooseNBGItem`, `trap_BotPushGoal`, `trap_BotPopGoal`, `trap_BotGetTopGoal`, `trap_AAS_*` (pathfinding, area queries, reachability)
- **sibling AI modules**: `BotChooseWeapon` (ai_dmq3.c), `BotRoamGoal` (ai_dmq3.c), `BotAlternateRoute`, `BotGoHarvest`, `BotWantsToRetreat`, `BotFindEnemy`, etc.
- **Chat/communication**: `BotAI_BotInitialChat`, `trap_BotEnterChat`, `BotVoiceChatOnly` (ai_chat.c)
- **Engine services**: `BotAI_Trace` (collision), `BotPointAreaNum` (AAS), `BotEntityInfo` (entity state), `trap_EA_*` (action interface)

## Design Patterns & Rationale

### State Machine (FSM) — Primary Pattern
The bot's behavior is a **12-state FSM**:
- `Intermission` → `Stand` → `Seek_LTG` (main loop)
- `Seek_LTG` ↔ `Seek_NBG` (short detours for items)
- `Seek_LTG` → `Battle_Fight` (enemy spotted)
- `Battle_Fight` ↔ `Battle_Chase`, `Battle_Retreat`, `Battle_NBG` (combat sub-states)
- `Seek_ActivateEntity` (button/trigger activation)
- `Respawn`, `Observer` (lifecycle states)

**Rationale**: FSMs were idiomatic for 2005 game AI. They are easy to debug (each state is a discrete code path), easy to interrupt (can transition on enemy spotting), and scale better than switch-statement dispatch. The `bs->ainode` function pointer pattern allows per-frame node execution without a central switch statement.

### Strategy Pattern — Secondary
`BotGetLongTermGoal` dispatches on `bs->ltgtype` and `gametype`:
- `LTG_TEAMHELP` → escort teammate
- `LTG_TEAMACCOMPANY` → follow teammate
- `LTG_DEFENDKEYAREA` → camp objective
- `LTG_KILL` → hunt nearest enemy
- (others for CTF, 1FCTF, Obelisk, Harvester)

Falls through to `BotGetItemLongTermGoal` if no type matches. **Rationale**: Keeps game-mode-specific logic encapsulated; enables reuse across modes.

### Adapter/Bridge Pattern
`ai_dmnet.c` bridges the **game logic layer** (what objectives exist) with the **navigation layer** (botlib). The `trap_BotLib*` syscall boundary is the contract; game VM never links to botlib directly—only calls through syscalls. **Rationale**: Allows botlib to be stateless (no knowledge of game rules) and reusable across different games/mods.

## Data Flow Through This File

**Input Sources:**
1. **Bot state** (`bot_state_t`): position, inventory, health, view angles, team, last-seen-enemy
2. **Entity queries** (`BotEntityInfo`, `BotEntityVisible`): teammate/enemy positions, visibility
3. **Navigation data** (via `trap_AAS_*`): reachability, travel times, swimming/ladder status
4. **Goal availability** (`trap_BotChooseLTGItem/NBGItem`): computed by botlib based on item locations, avoidance, fuzzy scoring

**Processing Pipeline:**
1. `BotResetNodeSwitches` (frame start)
2. For each bot: call `bs->ainode()` (current FSM state handler)
3. Node handler evaluates game state:
   - Is there a nearby goal? → push NBG
   - Can bot see enemy? → transition to Battle_Fight
   - Is teammate in trouble? → transition to LTG_TEAMHELP
   - Otherwise: update LTG via `BotGetLongTermGoal`
4. Move toward goal via `trap_BotMoveToGoal`
5. Output via `trap_EA_*` (movement, aiming, weapon, actions)

**Output Destinations:**
- `trap_EA_MoveForward`, `trap_EA_Jump`, `trap_EA_Crouch`: movement inputs
- `trap_EA_Attack`: weapon fire
- `trap_BotPushGoal`: navigation stack (botlib pathfinding)
- `trap_BotEnterChat`, voice chat: team communication
- `bs->ideal_viewangles`: view direction override

## Learning Notes

### Idiomatic to Quake III Era (2005)
- **Flat FSM instead of hierarchical state machines**: Easy to understand, no nesting complexity.
- **Global goal stack** (`trap_BotGetTopGoal`): Botlib manages a simple stack; no priority queues or dynamic re-planning.
- **No behavior trees**: Would have been clearer for complex routines like "accompany with formation and backup," but adds interpreter overhead.
- **Synchronous physics**: All AAS traces and pathfinding are real-time. Modern engines defer expensive queries to separate threads.
- **Scripted personality**: Bot behavior tweaked via fuzzy characteristic floats (crouching, camping, etc.) loaded from `chars.h`. Modern engines use learned policies or trained models.

### Modern Alternatives
- **ECS**: Replace `bot_state_t` monolith with component-based state (position, health, inventory, FSM_state, etc.)
- **Behavior trees / GOAP**: More flexible than FSM for complex multi-step tasks.
- **Planning**: Use a proper planner (HTN, STRIPS) instead of hard-coded LTG selection logic.
- **Async pathfinding**: Defer pathfinding to a separate thread/frame; use time-sliced goal evaluation.

### Key Insight: Separation of Concerns
The **game VM** asks "**what** should the bot do?"  
The **botlib** answers "**how** do I get there?"  
This separation—enforced by the syscall boundary—is the file's greatest architectural strength. It allowed the same botlib to be reused in multiple games (Q3A, RTCW, ET).

## Potential Issues

1. **Fixed node-switch buffer (144 bytes)**: The `nodeswitch[MAX_NODESWITCHES+1][144]` array allocates a tiny fixed buffer for debug logging. Format strings with long bot names or node names could overflow silently, corrupting the ring buffer or producing truncated traces.

2. **No error recovery on goal stack underflow**: `trap_BotPopGoal` and `trap_BotGetTopGoal` are called without visible null-checks. If the stack is corrupted or empty, the bot may hang or crash.

3. **Magic numbers**: Time constants (`6` seconds for air, `300` units for nearby goal, `400` units for formation distance) are scattered throughout, not configurable. Maps with extreme layouts might need tuning.

4. **Water/air survival is an interrupt but not preemptive**: `BotGoForAir` runs inside `BotGetLongTermGoal`, so if the bot is in Battle_Fight, it may not check air status frequently enough before drowning.

5. **Chat spam risk**: No visible rate-limiting on `BotAI_BotInitialChat` calls during team operations. A spamming bot could flood the server log or trigger spam filters.

6. **State explosion**: 12 FSM states × multiple LTG types × combat substates = many possible execution paths. Full state coverage testing is non-trivial.
