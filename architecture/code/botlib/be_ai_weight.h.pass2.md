# code/botlib/be_ai_weight.h — Enhanced Analysis

## Architectural Role

This header defines the **fuzzy weight configuration and evaluation system**, a core scoring subsystem within botlib's AI decision pipeline. It sits downstream of bot goal and movement FSMs (`be_ai_goal.c`, `be_ai_move.c`, `be_ai_weap.c`), providing the numerical scoring mechanism that converts bot inventory state into weighted priorities. The weights are **loaded once at bot initialization from external script files** and evaluated **every frame during goal/weapon/item evaluation**. This design decouples AI tuning from code, enabling offline genetic algorithm training of bot personalities without recompilation.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/botlib/be_ai_goal.c`** — Calls `FindFuzzyWeight` at init to locate named goal weights; calls `FuzzyWeight`/`FuzzyWeightUndecided` per-frame to score LTG/NBG goal desirability against bot's current inventory
- **`code/botlib/be_ai_move.c`** — Evaluates movement-related weights (e.g., evasion desirability based on nearby threats)
- **`code/botlib/be_ai_weap.c`** — Uses weights to select best weapon given current inventory and distance to target
- **`code/botlib/be_ai_char.c`** — Applies weight scaling to adjust bot personality traits (aggression, self-preservation) via `ScaleWeight`
- **`code/botlib/be_interface.c`** — Initializes weight config at `BotInitLoadWeights`; manages lifetime via `ReadWeightConfig`/`FreeWeightConfig`; calls `BotShutdownWeights` at engine shutdown

### Outgoing (what this file depends on)
- **`code/botlib/l_script.c`** / **`code/botlib/l_precomp.c`** — Used by `ReadWeightConfig` to parse `.weight` script files (fuzzy interval definitions)
- **`code/botlib/l_memory.c`** — Heap allocation for `weightconfig_t` structs and fuzzy separator trees
- **`code/botlib/l_libvar.c`** — Possibly for runtime weight tweaking/debugging via `libvar` key-value store
- **Engine collision/trace services** — Not directly, but weights score outcomes of reachability queries from `be_aas_reach.c` and movement predictions from `be_aas_move.c`

## Design Patterns & Rationale

### Fuzzy Interval Logic Tree
The `fuzzyseperator_t` linked-list node implements a **binary decision tree** where each node:
- Tests a bot `inventory[index]` value against a threshold (`value`)
- Returns a weighted score (`weight`) if the condition is met
- Recurses into `child` (true branch) or `next` (sibling/false branch)
- Enforces clamping via `minweight`/`maxweight` (prevents outlier tuning values)

**Rationale:** Fuzzy logic was a popular AI tuning approach in the early 2000s (late-90s quake/unreal era). Instead of hard thresholds ("if health <= 50, flee"), fuzzy weights assign continuous scores ("if health <= 50, assign weight 0.8 for flee goal"), allowing smooth, overlapping decision criteria. This avoids the "snapping" behavior of hard thresholds.

### Configuration-Driven Over Hardcoded
Weights live in **external script files** (loaded via `ReadWeightConfig`), not embedded in code. This enabled id Software to:
- Tune bot personalities post-release (balance updates)
- Use **offline genetic algorithms** (`EvolveWeightConfig`, `InterbreedWeightConfigs`) to evolve bots without recompilation
- Ship multiple weight configs for different difficulty levels / bot personalities

### Genetic Algorithm Support
The presence of `EvolveWeightConfig` (mutation) and `InterbreedWeightConfigs` (crossover) suggests the development team likely:
1. Generated base weight configs via manual tuning
2. Ran offline GA to evolve better bots against benchmark/self-play
3. Shipped the evolved weights in shipping maps/configs

This is a **non-real-time system** — evolution happens offline; the game runtime only *evaluates* weights.

## Data Flow Through This File

```
[Load Time]
  weight_config_file (e.g., "bots/heavy.weight")
    ↓
  ReadWeightConfig() — lexer → parser → fuzzy tree construction
    ↓
  weightconfig_t: name[] + fuzzyseperator_t* trees in memory
    ↓
  be_interface.c stores in botlib singleton

[Per-Frame Evaluation]
  bot.inventory[] (health, ammo, armor, powerups, flags, etc.)
    ↓
  be_ai_goal.c: FindFuzzyWeight(wc, "goal_ltg")
    ↓
  FuzzyWeight(inventory, wc, weight_index) — tree traversal + scoring
    ↓
  float weight_score (e.g., 0.0 = not desirable, 1.0 = very desirable)
    ↓
  Goal FSM uses score to prioritize LTG vs NBG vs item pickup
```

### Key State Transitions
- **Undecided boundaries:** `FuzzyWeightUndecided` variant handles the gray zone where separator conditions are ambiguous (e.g., health exactly at threshold). Exact semantics defined in `be_ai_weight.c`.
- **Personality scaling:** `ScaleWeight` / `ScaleBalanceRange` allow runtime adjustment of bot aggression/cowardice without reloading weights.

## Learning Notes

### Idiomatic to This Era
- **Fuzzy logic** for game AI was standard practice in 1999–2005 (Unreal, Doom 3 era)
- **Genetic algorithms for bot tuning** reflect the computational cost: runtime evaluation is *cheap* (tree traversal), but finding good weights offline is *hard* (hence GA)
- **Linked-list tree vs. static array:** The recursive structure (`child`/`next` pointers) is flexible but risks deep stacks if configs have long chains

### Modern Contrast
Today's engines often use:
- **Neural networks + reinforcement learning** for bot behavior (learn end-to-end)
- **Behavior trees** with parameterized nodes rather than fuzzy trees
- **Runtime tuning** via online learning, not offline GA

### Connection to Game Engine Concepts
- **Fuzzy logic** is a form of **weighted decision-making** (ancestor of modern utility-based AI)
- **Genetic algorithms** are **meta-heuristics** for hyperparameter tuning (modern analog: gradient descent + loss functions)
- **Separation of config from code** is **data-driven design** (now universal best practice)

## Potential Issues

### Code Quality / Stability
1. **Struct name typo:** `fuzzyseperator_t` should be `fuzzyseparator_t` — misspelling is embedded in the codebase; unlikely to be fixed without breaking binary compatibility
2. **Recursive tree depth risk:** Malformed or deeply-nested weight configs could cause stack overflow. No apparent depth checks in `FuzzyWeight` tree traversal.
3. **Hard limit:** `MAX_WEIGHTS = 128` is a static ceiling; if bots ever needed >128 named weight functions, the array would overflow.

### Architectural Tradeoffs
- **Static array in `weightconfig_t`** (`weight_t weights[MAX_WEIGHTS]`) wastes memory if only a few weights are used; no dynamic allocation alternative provided
- **No bounds validation** on `inventory[index]` in `FuzzyWeight` — caller must ensure `index` is valid or undefined behavior occurs
- **No caching of fuzzy evaluation results** — each goal evaluation re-traverses the tree; if a bot has 50 goals, that's 50 tree walks per frame (likely acceptable for 2005 CPUs, but not optimal)

### Cross-Cutting Observations
- **Tight coupling to botlib script parser:** Weight configs are **not portable** to other engines without duplicating `l_script.c` + `l_precomp.c`
- **No versioning in binary weight format:** If the `aasfile_t` binary header version changes, old weight files may silently fail to load (cf. `AAS_DumpAASData` with explicit version fields)
