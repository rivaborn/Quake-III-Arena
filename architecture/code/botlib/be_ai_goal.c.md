# code/botlib/be_ai_goal.c

## File Purpose
Implements the bot goal AI subsystem for Quake III Arena. It manages level item tracking, per-bot goal stacks, and fuzzy-weight-based goal selection (both long-term and nearby goals) to drive bot navigation decisions.

## Core Responsibilities
- Load and manage item configuration (`items.c`) describing all pickup types in the level
- Build and maintain a runtime list of `levelitem_t` instances from BSP entities and live entity state
- Provide per-bot goal stacks (push/pop/query) via opaque integer handles
- Maintain per-bot avoid-goal lists with expiry times to prevent re-targeting recently visited items
- Select the best Long-Term Goal (LTG) and Near-By Goal (NBG) using fuzzy weight scoring divided by AAS travel time
- Parse `target_location` and `info_camp` BSP entities into map location and camp spot lists
- Track dynamically dropped entity items with timeout-based expiry

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `maplocation_t` | struct | Stores a named map location (`target_location`) with origin and AAS area number |
| `campspot_t` | struct | Stores a named camp spot (`info_camp`) with origin, area, range, weight, wait, random |
| `gametype_t` | enum | Game type constants (FFA, Tournament, Single Player, Team, CTF, …) |
| `levelitem_t` | struct | A runtime pickup instance; tracks entity linkage, goal area, timeout, flags, and weight |
| `iteminfo_t` | struct | Static descriptor for a pickup class: classname, model, type, inventory index, respawn time, bounds |
| `itemconfig_t` | struct | Array of all `iteminfo_t` entries loaded from `items.c` |
| `bot_goalstate_t` | struct | Per-bot state: goal stack, avoid lists, weight config/index, client ID, last reachability area |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botgoalstates` | `bot_goalstate_t *[MAX_CLIENTS+1]` | global | Per-client goal state handles indexed 1..MAX_CLIENTS |
| `itemconfig` | `itemconfig_t *` | global | Singleton loaded item configuration |
| `levelitemheap` | `levelitem_t *` | global | Pre-allocated pool of level item nodes |
| `freelevelitems` | `levelitem_t *` | global | Head of free list within `levelitemheap` |
| `levelitems` | `levelitem_t *` | global | Head of active doubly-linked level item list |
| `numlevelitems` | `int` | global | Running count of level items ever allocated |
| `maplocations` | `maplocation_t *` | global | Singly-linked list of parsed map locations |
| `campspots` | `campspot_t *` | global | Singly-linked list of parsed camp spots |
| `g_gametype` | `int` | global | Current game type, controls item flag filtering |
| `droppedweight` | `libvar_t *` | global | Configurable bonus weight for dropped items |

## Key Functions / Methods

### BotSetupGoalAI
- **Signature:** `int BotSetupGoalAI(void)`
- **Purpose:** Initializes the entire goal AI subsystem at map load.
- **Inputs:** None (reads `g_gametype` and `itemconfig` libvars).
- **Outputs/Return:** `BLERR_NOERROR` or `BLERR_CANNOTLOADITEMCONFIG`.
- **Side effects:** Allocates `itemconfig` from hunk; sets global `g_gametype` and `droppedweight`.
- **Calls:** `LibVarValue`, `LibVarString`, `LibVar`, `LoadItemConfig`, `botimport.Print`.

### BotShutdownGoalAI
- **Signature:** `void BotShutdownGoalAI(void)`
- **Purpose:** Tears down all goal AI state; frees item config, level item heap, info entities, and all bot goal states.
- **Side effects:** Frees all global pointers; calls `BotFreeGoalState` per client.

### LoadItemConfig
- **Signature:** `itemconfig_t *LoadItemConfig(char *filename)`
- **Purpose:** Parses `items.c` script into an `itemconfig_t` allocated on the hunk.
- **Inputs:** Config filename string.
- **Outputs/Return:** Pointer to loaded `itemconfig_t`, or `NULL` on error.
- **Side effects:** Hunk allocation; reads file via `LoadSourceFile`/`PC_ReadToken`.
- **Calls:** `LibVarValue`, `LoadSourceFile`, `GetClearedHunkMemory`, `PC_ReadToken`, `ReadStructure`, `FreeSource`, `FreeMemory`.

### BotInitLevelItems
- **Signature:** `void BotInitLevelItems(void)`
- **Purpose:** Populates the active level item list from all BSP entities at map start.
- **Side effects:** Calls `BotInitInfoEntities`, `InitLevelItemHeap`, iterates BSP entities, drops items to floor, resolves goal areas. Modifies `levelitems`, `numlevelitems`.
- **Calls:** `AAS_NextBSPEntity`, `AAS_VectorForBSPEpairKey`, `AAS_DropToFloor`, `AAS_BestReachableArea`, `AAS_BestReachableFromJumpPadArea`, `AAS_Trace`, `AAS_PointContents`, `AllocLevelItem`, `AddLevelItemToList`.

### BotUpdateEntityItems
- **Signature:** `void BotUpdateEntityItems(void)`
- **Purpose:** Per-frame update: times out expired dropped items and links/registers new entity items found in AAS entity list.
- **Side effects:** Mutates `levelitems` list (removes timed-out, adds new dropped items). Updates goal area numbers.
- **Calls:** `AAS_NextEntity`, `AAS_EntityType`, `AAS_EntityModelindex`, `AAS_EntityInfo`, `AAS_BestReachableArea`, `AAS_AreaJumpPad`, `AAS_Time`, `RemoveLevelItemFromList`, `FreeLevelItem`, `AllocLevelItem`, `AddLevelItemToList`.

### BotChooseLTGItem
- **Signature:** `int BotChooseLTGItem(int goalstate, vec3_t origin, int *inventory, int travelflags)`
- **Purpose:** Selects the highest-value long-term goal item via fuzzy weight / travel-time scoring and pushes it onto the goal stack.
- **Inputs:** Goal state handle, bot world origin, bot inventory array, allowed travel flags.
- **Outputs/Return:** `qtrue` if a goal was chosen and pushed; `qfalse` otherwise.
- **Side effects:** Calls `BotAddToAvoidGoals`, `BotPushGoal`.
- **Calls:** `BotReachabilityArea`, `AAS_AreaReachability`, `AAS_AreaTravelTimeToGoalArea`, `FuzzyWeight` / `FuzzyWeightUndecided`, `BotAvoidGoalTime`, `BotAddToAvoidGoals`, `BotPushGoal`.

### BotChooseNBGItem
- **Signature:** `int BotChooseNBGItem(int goalstate, vec3_t origin, int *inventory, int travelflags, bot_goal_t *ltg, float maxtime)`
- **Purpose:** Selects the best nearby goal item reachable within `maxtime` that doesn't detour excessively from the long-term goal.
- **Inputs:** As above plus existing LTG and max travel-time budget.
- **Outputs/Return:** `qtrue` on success; `qfalse` otherwise.
- **Side effects:** Same as `BotChooseLTGItem`.
- **Notes:** Uses `ltg_time` to reject items whose detour would exceed remaining LTG travel.

### BotTouchingGoal
- **Signature:** `int BotTouchingGoal(vec3_t origin, bot_goal_t *goal)`
- **Purpose:** AABB test to determine if the bot's position overlaps a goal's bounding box.
- **Calls:** `AAS_PresenceTypeBoundingBox`.

### BotItemGoalInVisButNotVisible
- **Signature:** `int BotItemGoalInVisButNotVisible(int viewer, vec3_t eye, vec3_t viewangles, bot_goal_t *goal)`
- **Purpose:** Detects if an item goal is in the PVS but its entity is stale (not updated recently), indicating it has been picked up.
- **Calls:** `AAS_Trace`, `AAS_EntityInfo`, `AAS_Time`.

### Notes
- Trivial heap management helpers (`AllocLevelItem`, `FreeLevelItem`, `AddLevelItemToList`, `RemoveLevelItemFromList`, `InitLevelItemHeap`) form a fixed-size pool allocator.
- `BotPushGoal`, `BotPopGoal`, `BotEmptyGoalStack`, `BotGetTopGoal`, `BotGetSecondGoal` are thin wrappers around the per-bot `goalstack[]` array.
- `BotAddToAvoidGoals`, `BotRemoveFromAvoidGoals`, `BotAvoidGoalTime`, `BotSetAvoidGoalTime` manage the fixed-size avoid list with AAS-time-based expiry.

## Control Flow Notes
- **Init:** `BotSetupGoalAI` → `LoadItemConfig`. Called once at subsystem startup.
- **Map load:** `BotInitLevelItems` → `BotInitInfoEntities` (parses BSP info entities) + `InitLevelItemHeap` + BSP entity scan. Called per-map.
- **Per-frame:** `BotUpdateEntityItems` reconciles the live entity list with the level item list (called by the game each frame).
- **Per-bot think:** `BotChooseLTGItem` / `BotChooseNBGItem` are called by the AI decision layer to push goals; movement code queries the stack via `BotGetTopGoal`.
- **Shutdown:** `BotShutdownGoalAI` frees all resources.

## External Dependencies
- `q_shared.h` — `vec3_t`, `qboolean`, `Com_Memset/Memcpy`, string utilities
- `l_libvar.h` — `LibVar`, `LibVarValue`, `LibVarString` (botlib config variables)
- `l_memory.h` — `GetClearedMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `l_log.h` — `Log_Write` (diagnostic logging)
- `l_script.h` / `l_precomp.h` — `LoadSourceFile`, `PC_ReadToken`, `PC_ExpectTokenType`, `FreeSource`, `SourceError`
- `l_struct.h` — `ReadStructure` (struct-driven config parsing)
- `be_aas_funcs.h` / `be_aas.h` — AAS queries (`AAS_AreaTravelTimeToGoalArea`, `AAS_BestReachableArea`, `AAS_PointAreaNum`, `AAS_Trace`, `AAS_NextBSPEntity`, `AAS_NextEntity`, `AAS_EntityInfo`, `AAS_Time`, etc.) — **defined in AAS subsystem**
- `be_ai_weight.h` — `FuzzyWeight`, `FuzzyWeightUndecided`, `ReadWeightConfig`, `FreeWeightConfig`, `FindFuzzyWeight`, `InterbreedWeightConfigs`, `EvolveWeightConfig` — **defined in be_ai_weight.c**
- `be_interface.h` — `botimport` (engine import struct for Print), `bot_developer` — **defined in be_interface.c**
- `be_ai_move.h` — `BotReachabilityArea` — **defined in be_ai_move.c**
- `be_ai_goal.h` — public API declarations (`bot_goal_t`, `MAX_GOALSTACK`, `MAX_AVOIDGOALS`, `GFL_*` flags, `BLERR_*` error codes)
