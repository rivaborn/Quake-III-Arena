# code/botlib/be_ai_goal.c — Enhanced Analysis

## Architectural Role

`be_ai_goal.c` occupies the decision layer of the botlib pipeline, sitting between the raw AAS spatial database (which knows *how* to move) and the movement execution layer (which knows *how to physically execute* a path). It answers the question "where should the bot go next?" by maintaining per-bot goal stacks and scoring candidate level items using fuzzy weights divided by AAS travel time. This file is the primary consumer of both `be_aas_route.c` (travel time queries) and `be_ai_weight.c` (fuzzy weight evaluation), and its output — the `bot_goal_t` at the top of each bot's stack — feeds directly into `be_ai_move.c`'s travel-type execution FSM. It also maintains the only runtime map of all pickup entities, making it the authoritative item-tracking layer for the entire bot subsystem.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/botlib/be_interface.c`** — exports every public function (`BotChooseLTGItem`, `BotChooseNBGItem`, `BotGetTopGoal`, `BotTouchingGoal`, `BotAllocGoalState`, `BotFreeGoalState`, `BotInitLevelItems`, `BotUpdateEntityItems`, `BotSetupGoalAI`, `BotShutdownGoalAI`, etc.) through the `botlib_export_t` vtable returned by `GetBotLibAPI`. This is the only path by which the engine reaches goal logic.
- **`code/game/g_bot.c`** — calls `BotAllocGoalState`/`BotFreeGoalState` and `BotSetupGoalAI`/`BotShutdownGoalAI` through `trap_BotLib*` syscalls during bot spawn/removal and map load/unload.
- **`code/game/ai_main.c`** / **`ai_dmq3.c`** / **`ai_dmnet.c`** — the per-bot FSM calls `BotChooseLTGItem`, `BotChooseNBGItem`, `BotGetTopGoal`, `BotGetSecondGoal`, `BotTouchingGoal`, `BotPushGoal`, `BotPopGoal`, and `BotItemGoalInVisButNotVisible` each AI think cycle. This is the functional consumer of the goal stack.
- **`code/server/sv_bot.c`** — drives `BotUpdateEntityItems` and `BotInitLevelItems` calls each server frame and map load, via `trap_BotLib*`.

### Outgoing (what this file depends on)

- **`be_aas_route.c`** (`AAS_AreaTravelTimeToGoalArea`) — the core denominator in the fuzzy scoring formula `weight / traveltime`. Every LTG/NBG candidate evaluation makes this call.
- **`be_aas_sample.c`** (`AAS_PresenceTypeBoundingBox`, `AAS_PointAreaNum`, `AAS_BestReachableArea`, `AAS_TraceAreas`, `AAS_AreaReachability`) — used during item initialization to resolve goal areas and during `BotTouchingGoal` for AABB tests.
- **`be_aas_bspq3.c`** (`AAS_NextBSPEntity`, `AAS_ValueForBSPEpairKey`, `AAS_VectorForBSPEpairKey`, `AAS_FloatForBSPEpairKey`, `AAS_Trace`) — BSP entity iteration and epair extraction during map init; trace during `BotItemGoalInVisButNotVisible`.
- **`be_aas_entity.c`** (`AAS_NextEntity`, `AAS_EntityType`, `AAS_EntityModelindex`, `AAS_EntityInfo`) — per-frame entity state polling in `BotUpdateEntityItems`.
- **`be_aas_reach.c`** (`AAS_BestReachableFromJumpPadArea`, `AAS_AreaReachability`) — jump-pad and reachability resolution during item initialization and LTG scoring.
- **`be_aas_move.c`** (`AAS_DropToFloor`, `BotReachabilityArea`) — floor-snapping items to valid geometry and finding the bot's current reachability area.
- **`be_aas_main.c`** (`AAS_Time`) — monotonic AAS time used for dropped-item timeout and entity staleness detection.
- **`be_ai_weight.c`** (`FuzzyWeight`, `FuzzyWeightUndecided`, `ReadWeightConfig`, `FreeWeightConfig`, `FindFuzzyWeight`, `InterbreedWeightConfigs`, `EvolveWeightConfig`) — the fuzzy evaluation engine; the weight half of `weight / traveltime`.
- **`be_interface.c`** — reads `botimport` (for `Print`, `Trace`, `EntityInfo`, etc.) and `bot_developer` flag for verbose logging.
- **`l_libvar.c`** — reads `g_gametype`, `droppedweight`, `max_iteminfo`, `max_levelitems` as runtime config variables, decoupling from compile-time constants.
- **`l_precomp.c`** / **`l_script.c`** / **`l_struct.c`** — drives `LoadItemConfig` parsing of the `items.c` data file.

## Design Patterns & Rationale

- **Handle-based API over raw pointers:** `botgoalstates[handle]` maps an integer client index to a heap-allocated `bot_goalstate_t`. This isolates the game VM (which only holds opaque integers) from botlib's internal representation, and is consistent across the entire botlib AI layer (`be_ai_move.c`, `be_ai_chat.c`, etc.).
- **Fixed-size pool allocator for `levelitem_t`:** `InitLevelItemHeap` pre-allocates a contiguous block and threads it into a free list. This avoids fragmentation during the per-frame `BotUpdateEntityItems` add/remove cycle and keeps allocation O(1). The pool size is driven by `max_levelitems` libvar so it can be tuned without recompilation.
- **Fuzzy value/time scoring:** Rather than a Euclidean distance heuristic, the bot divides fuzzy item weight (derived from the bot's current inventory state) by AAS area travel time. This naturally prefers nearby high-value items and far-away items only when they are worth the detour — an elegant fusion of AI desirability with pathfinding cost.
- **Static config + dynamic instances:** `itemconfig_t` (from `items.c`) is loaded once per subsystem init and never mutated; `levelitem_t` instances are created per-map and mutated per-frame. This clean separation prevents the heavy script parsing from being repeated and allows the dynamic list to be rapidly rebuilt.
- **Avoid list with time-based expiry:** Rather than tracking "did I pick this up?" (which requires game-side feedback), the bot simply avoids re-targeting a goal for `AVOID_DEFAULT_TIME` AAS seconds after pursuing it. This is an empirical hack that works well enough for arena pickup cycles (respawn times are predictable) without requiring tight coupling to the game VM.

## Data Flow Through This File

```
BSP entities (static, map load)
    → BotInitInfoEntities()   → maplocations, campspots linked lists
    → BotInitLevelItems()     → levelitems doubly-linked list (levelitem_t per pickup)

Live entity stream (per frame)
    → BotUpdateEntityItems()  → adds dropped/new items; removes timed-out items

Per-bot think (ai_dmnet FSM)
    bot origin + inventory + travelflags
    → BotChooseLTGItem()
        → BotReachabilityArea()          (bot's current area)
        → for each levelitem:
            FuzzyWeight(inventory)       (desirability)
            AAS_AreaTravelTimeToGoalArea  (cost)
            score = weight / (1 + time * TRAVELTIME_SCALE)
        → BotPushGoal(best)             → goalstack[top++]

    → BotChooseNBGItem()
        → same scoring, but also bounds-checks travel vs LTG detour cost
        → BotPushGoal(best)             → goalstack[top++]

    → BotGetTopGoal()                   → be_ai_move.c reads goal
    → BotTouchingGoal()                 → triggers goal pop when reached
    → BotPopGoal()                      → goalstack[top--]
```

The avoid list forms a side channel: after pushing a goal, `BotAddToAvoidGoals` records it with an expiry timestamp so the same item isn't immediately re-targeted next think.

## Learning Notes

- **Fuzzy logic AI in 1999:** This predates GOAP (Goal-Oriented Action Planning, popularized by F.E.A.R. in 2005) and behavior trees. The approach is hand-coded utility scoring — a precursor to what is now called "utility AI." The `be_ai_weight.c` fuzzy system with its `#ifdef UNDECIDEDFUZZY` / `#ifdef RANDOMIZE` paths hints at experimentation with non-deterministic evaluation to produce more varied bot behavior.
- **AAS travel time as a cost metric:** Modern engines use navmesh with configurable edge costs. Q3's AAS stores area-to-area travel times pre-computed by Dijkstra across the cluster hierarchy. Using these as denominator (not Euclidean distance) means bots correctly account for geometry — a long corridor costs more than an open room of the same Euclidean distance.
- **Goal stack vs. behavior tree:** The two-level LTG/NBG stack is a very early form of hierarchical goal decomposition. The NBG is essentially an opportunistic interrupt: "pursue the long-term goal, but grab this nearby health if the detour is cheap." Modern engines express this as interrupting behavior tree branches.
- **Game type filtering via `IFL_*` flags:** The `IFL_NOTFREE`, `IFL_NOTTEAM`, `IFL_NOTSINGLE` flags on `levelitem_t` mirror the game's item flag system but are maintained independently in botlib. The `g_gametype` global is read from a libvar — it must be explicitly set by the game VM and is never automatically synchronized, a fragile coupling pattern.
- **`FIXME: these are game specific`:** The embedded `gametype_t` enum (line ~87) duplicates the one in `code/game/g_local.h` with no shared header. Any game-type addition in the game VM must be manually mirrored here — a maintenance hazard that modern engines avoid with shared protocol headers or code generation.

## Potential Issues

- **O(items × bots) per frame:** Both `BotChooseLTGItem` and `BotChooseNBGItem` scan the full `levelitems` linked list for every bot every think cycle. With 16 bots and 256 items this is 4096 AAS travel-time queries per frame — each of which may trigger a cluster routing cache lookup. This was acceptable on 1999 hardware with typical Q3 map item counts (~60–120 items) but would degrade on maps with many more entities.
- **`botgoalstates` uninitialized:** The `bk001206 - FIXME: init?` comment on the `botgoalstates` array declaration indicates it was flagged at the 2001 bugfix pass. The array is in BSS (zero-initialized at program start), so NULL checks in `BotGoalStateFromHandle` protect against uninitialized-slot access — but this is implicit, not explicit.
- **Avoid list linear search:** `BotAvoidGoalTime` and `BotAddToAvoidGoals` both scan the fixed `MAX_AVOIDGOALS` array linearly. Given `MAX_AVOIDGOALS` is likely small (< 64), this is negligible in practice but architecturally inelegant.
