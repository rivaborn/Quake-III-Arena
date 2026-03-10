# code/game/be_ai_goal.h

## File Purpose
Public interface header for the bot goal AI subsystem in Quake III Arena's botlib. It defines the `bot_goal_t` structure and declares all functions used to manage bot goals, goal stacks, item weights, and fuzzy logic for goal selection.

## Core Responsibilities
- Define the `bot_goal_t` structure representing a navigable destination
- Declare goal state lifecycle management (alloc, reset, free)
- Declare goal stack push/pop/query operations
- Declare long-term goal (LTG) and nearby goal (NBG) item selection
- Declare item weight loading and fuzzy logic mutation/interbreeding
- Declare level item initialization and dynamic entity item updates
- Declare avoid-goal tracking and timing

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bot_goal_t` | struct (typedef) | Represents a single bot goal: world position, bounding box, entity reference, classification flags, and item metadata |

## Global / File-Static State
None — header only; no definitions.

## Key Functions / Methods

### BotChooseLTGItem
- Signature: `int BotChooseLTGItem(int goalstate, vec3_t origin, int *inventory, int travelflags)`
- Purpose: Selects the best long-term goal item for the bot using fuzzy logic and item weights
- Inputs: Goal state handle, current bot origin, inventory array, AAS travel flags
- Outputs/Return: Non-zero if a suitable LTG was found
- Side effects: Writes chosen goal into the goal state internally
- Calls: Defined in `be_ai_goal.c` (botlib)
- Notes: Primary goal-selection entry point for strategic navigation

### BotChooseNBGItem
- Signature: `int BotChooseNBGItem(int goalstate, vec3_t origin, int *inventory, int travelflags, bot_goal_t *ltg, float maxtime)`
- Purpose: Selects a nearby goal that doesn't significantly detour the bot from its LTG
- Inputs: Goal state, current origin, inventory, travel flags, current LTG, max acceptable detour time
- Outputs/Return: Non-zero if a valid NBG was found
- Side effects: None externally visible
- Notes: Travel time from NBG to LTG must not exceed travel time from current position to LTG

### BotPushGoal / BotPopGoal / BotEmptyGoalStack
- Signature: `void BotPushGoal(int goalstate, bot_goal_t *goal)` / `void BotPopGoal(int goalstate)` / `void BotEmptyGoalStack(int goalstate)`
- Purpose: Manage a LIFO goal stack (max depth `MAX_GOALSTACK` = 8) per bot
- Inputs: Goal state handle; push also takes a goal pointer
- Side effects: Modifies internal goal stack for the given state

### BotGetTopGoal / BotGetSecondGoal
- Signature: `int BotGetTopGoal(int goalstate, bot_goal_t *goal)` / `int BotGetSecondGoal(int goalstate, bot_goal_t *goal)`
- Purpose: Peek at the top or second goal without popping
- Outputs/Return: Non-zero if a goal exists at that stack position; fills `*goal`

### BotAllocGoalState / BotFreeGoalState
- Signature: `int BotAllocGoalState(int client)` / `void BotFreeGoalState(int handle)`
- Purpose: Create and destroy per-bot goal state objects tied to a client slot
- Outputs/Return: Handle to the allocated state

### BotSetupGoalAI / BotShutdownGoalAI
- Signature: `int BotSetupGoalAI(void)` / `void BotShutdownGoalAI(void)`
- Purpose: Subsystem init/shutdown; called once at botlib startup/teardown
- Side effects: Allocates/frees global goal AI resources

### BotInitLevelItems / BotUpdateEntityItems
- Signature: `void BotInitLevelItems(void)` / `void BotUpdateEntityItems(void)`
- Purpose: Scan and register static level items at map load; periodically refresh dynamic entities (dropped weapons, CTF flags)
- Side effects: Modifies internal item goal database

### BotAvoidGoalTime / BotSetAvoidGoalTime
- Signature: `float BotAvoidGoalTime(int goalstate, int number)` / `void BotSetAvoidGoalTime(int goalstate, int number, float avoidtime)`
- Purpose: Get/set cooldown duration preventing the bot from re-pursuing a recently visited goal (up to `MAX_AVOIDGOALS` = 256 entries)

### Notes
- `BotLoadItemWeights` / `BotFreeItemWeights`: Load fuzzy logic item weight files (`.c` scripts) that drive LTG scoring; freed on bot disconnect.
- `BotInterbreedGoalFuzzyLogic` / `BotMutateGoalFuzzyLogic` / `BotSaveGoalFuzzyLogic`: Genetic algorithm hooks for evolving bot goal weights offline; not used in normal gameplay.
- `BotTouchingGoal`: Spatial overlap test between bot origin and goal AABB.
- `BotItemGoalInVisButNotVisible`: Detects when a visible-but-missing item indicates it was recently picked up.

## Control Flow Notes
- `BotSetupGoalAI` is called during botlib initialization.
- `BotInitLevelItems` is called at map load; `BotUpdateEntityItems` is called each server frame.
- Per-bot: `BotAllocGoalState` at bot spawn → `BotLoadItemWeights` → per-frame `BotChooseLTGItem`/`BotChooseNBGItem` → stack push/pop → `BotFreeItemWeights` + `BotFreeGoalState` at disconnect.
- `BotShutdownGoalAI` is called at botlib shutdown.

## External Dependencies
- `vec3_t` — defined in `q_shared.h`
- `MAX_AVOIDGOALS`, `MAX_GOALSTACK`, `GFL_*` flags — defined in this file
- All function bodies — defined in `code/botlib/be_ai_goal.c`
- AAS travel flag constants (`travelflags`) — defined in `be_aas.h` / `aasfile.h`
