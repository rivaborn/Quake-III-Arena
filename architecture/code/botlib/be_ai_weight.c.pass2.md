# code/botlib/be_ai_weight.c — Enhanced Analysis

## Architectural Role

This file is the **decision-scoring engine** for the botlib subsystem, implementing fuzzy logic evaluation that translates bot state (inventory, loadout) into numerical weights for goal selection, weapon choice, and behavior. It sits in the critical path between bot decision modules (`be_ai_goal.c`, `be_ai_weap.c`) and the actual action execution, enabling bots to make probabilistic and deterministic choices driven by declaratively-configured weight hierarchies. The file also implements genetic mutation (`EvolveWeightConfig`) and blending (`InterbreedWeightConfigs`) to support offline bot difficulty tuning and skill progression.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/ai_main.c` / `code/game/ai_dmq3.c`**: The game VM's bot FSM calls into botlib to score goals and select weapons via `trap_BotLib*` syscalls, which eventually invoke `FuzzyWeight*` functions in this file via `be_interface.c`
- **`code/botlib/be_ai_goal.c`**: Goal/LTG/NBG scoring uses `FuzzyWeight` to evaluate candidate goals based on inventory and distance
- **`code/botlib/be_ai_weap.c`**: Weapon selection directly calls `FuzzyWeight*` to rank available weapons by inventory state
- **`code/botlib/be_ai_main.c`**: Bot initialization parses weight configs via `ReadWeightConfig`; shutdown calls `BotShutdownWeights`
- **`code/botlib/be_interface.c`**: The botlib→game bridge exposes weight functions through the `botlib_export_t` vtable (e.g., `BotLoadWeights`, `BotEvolveWeights`)

### Outgoing (what this file depends on)
- **`code/botlib/l_script.h` / `l_precomp.h`**: Source file parsing (`PC_*` token functions, `LoadSourceFile`, `FreeSource`)
- **`code/botlib/l_memory.h`**: All heap allocation (`GetClearedMemory`, `FreeMemory`)
- **`code/botlib/l_libvar.h`**: Reads `bot_reloadcharacters` cvar to control caching behavior
- **`code/botlib/be_interface.h`**: Uses `botimport` callbacks for file I/O and error reporting
- **`code/game/q_shared.h`**: Random number generation (`random()`, `crandom()`), math types, string utilities

## Design Patterns & Rationale

**Hierarchical Switch/Case Tree with Recursive Evaluation**
- Each `fuzzyseperator_t` node represents a switch statement on an inventory index (e.g., `switch(INVENTORY_ARMOR)`)
- Child nodes allow nested switches, enabling complex multi-condition decisions (e.g., "if health < 50, then consider X based on ammo; else consider Y based on armor")
- `ReadFuzzySeperators_r` recursively parses and builds this tree; `FuzzyWeight_r` recursively evaluates it
- **Rationale**: Declarative, compile-time-static decision hierarchies avoid hardcoded bot behavior; trees are human-readable and editable

**File-Level Caching with LRU Eviction**
- `weightFileList[MAX_WEIGHT_FILES]` holds parsed configs; checked before disk reload if `bot_reloadcharacters` is off
- **Rationale**: Avoids repeated parsing of the same weight file for multiple bots; speeds up bot spawning in large matches

**Linear Interpolation Between Case Boundaries**
- When inventory falls between two case values, `FuzzyWeight_r` linearly interpolates between adjacent case weights
- **Rationale**: Smooth, continuous weight curves; avoids discontinuities that could cause abrupt behavior changes

**Deterministic vs. Stochastic Duality** (`FuzzyWeight` vs. `FuzzyWeightUndecided`)
- `FuzzyWeight_r` returns exact interpolated weight; `FuzzyWeightUndecided_r` randomly samples `[minweight, maxweight]`
- **Rationale**: Deterministic for critical decisions (navigation safety); stochastic for goal/weapon selection to create varied bot personalities

**Genetic Algorithm Support**
- `EvolveWeightConfig` mutates `WT_BALANCE` weights (1% reroll, 50% Gaussian step, rest unchanged)
- `InterbreedWeightConfigs` averages two parent configs into a child
- **Rationale**: Supports offline difficulty tuning and bot breeding for skill progression without code changes

## Data Flow Through This File

1. **Initialization Phase** (`be_ai_main.c` → `ReadWeightConfig`):
   - Game loads bot character, e.g., "mynbot.c"
   - `ReadWeightConfig` checks `weightFileList` cache; if miss, parses file via script parser
   - Each `weight` declaration becomes a `weight_t` entry; each `switch` statement becomes a `fuzzyseperator_t` tree
   - Config cached in `weightFileList` slot (if `bot_reloadcharacters == 0`)

2. **Decision-Scoring Phase** (per-frame, from `be_ai_goal.c` / `be_ai_weap.c`):
   - Bot's `inventory[]` array is passed to `FuzzyWeight(inventory, config, weightnum)`
   - Tree traversal: switch on inventory index, find matching case/default, interpolate or recurse
   - Returns float score; higher = better fit for this decision

3. **Mutation Phase** (offline bot training):
   - `EvolveWeightConfig` walks all `WT_BALANCE` leaf nodes, perturbs weights (mutation)
   - `InterbreedWeightConfigs` blends two parent configs, component-wise, for genetic diversity
   - Modified config saved back to disk (by external code)

4. **Cleanup** (`BotShutdownWeights`):
   - Walks `weightFileList`, frees all parsed trees via `FreeFuzzySeperators_r`, clears slots

## Learning Notes

**Idiomatic to Early-2000s Game AI:**
- Fuzzy logic was a popular AI technique pre-planning systems; it's more declarative than pure code but less flexible than modern behavior trees or planners
- The weight-file format mirrors early AI middleware (e.g., FEAR's fuzzy logic editor)
- No dynamic rebalancing; weights are static per-session (mutation happens offline)

**Connection to Engine Concepts:**
- **Separation of Data and Logic**: Weight configs are data (declarative `.c` files); evaluation is generic
- **Caching Discipline**: Mirrors engine-wide patterns in `Renderer` (shader cache), `Client` (model/sound cache)
- **Genetic Algorithm Integration**: Precursor to modern machine-learning-based difficulty tuning; still used in some games for procedural bot variety

**Modern Alternatives:**
- Behavior trees or hierarchical task networks (HTN) for more complex composition
- Neural networks or reinforcement learning for data-driven bot behavior
- Dynamic weight rebalancing based on win/loss metrics

**Key Insight for Developers:**
The `switch(inventory_index)` tree structure is essentially a decision tree flattened into explicit case statements. Understanding how `FuzzyWeight_r` traverses and interpolates is key to tweaking bot difficulty without code changes.

## Potential Issues

- **No Validation of Weight Config Consistency**: If two inherited weight configs have mismatched tree structure, `InterbreedWeightConfigs` silently skips mismatched nodes or logs errors. This can silently degrade interbreeding results.
- **No Bounds Checking on Inventory Array Access**: `FuzzyWeight_r` accesses `inventory[fs->index]` without verifying `fs->index` is valid for the bot's inventory size.
- **Linear Interpolation Edge Case**: If a case boundary is exactly at an inventory value, interpolation could mathematically produce values outside `[minweight, maxweight]` due to floating-point precision (though unlikely in practice).
- **Caching Race Condition**: If `bot_reloadcharacters` is toggled at runtime while bots are active, the cache behavior becomes inconsistent (some bots see old configs, others see new ones).
- **Memory Leak on Parse Error**: `ReadFuzzySeperators_r` attempts to free partial trees on error, but if an error occurs late in recursion, the caller (`ReadWeightConfig`) may still leak if `FreeWeightConfig` doesn't fully clean up in all error paths.
