# code/botlib/be_ai_weight.h

## File Purpose
Defines the data structures and public API for the botlib's fuzzy logic weighting system. It provides a configuration-driven framework for evaluating weighted decisions based on bot inventory state, used by the AI goal and decision-making subsystems.

## Core Responsibilities
- Define the `fuzzyseperator_t` linked-list node for fuzzy logic interval separation
- Define `weight_t` (named weight entry) and `weightconfig_t` (full weight configuration) containers
- Declare I/O functions for loading, saving, and freeing weight configurations from disk
- Declare evaluation functions that compute fuzzy weight values from bot inventory
- Declare mutation/evolution utilities for weight configs (used in bot training/genetic-style tuning)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `fuzzyseperator_t` | struct (recursive linked list node) | Represents one interval/branch in a fuzzy decision tree; holds `index` into inventory, `value` threshold, `type`, scalar `weight`, clamped `minweight`/`maxweight`, and pointers to `child` (sub-tree) and `next` (sibling) |
| `weight_t` | struct | Named weight entry: a `name` string and pointer to the root `fuzzyseperator_t` chain |
| `weightconfig_t` | struct | Full weight configuration: count of weights, fixed array of up to `MAX_WEIGHTS` `weight_t` entries, and source `filename` |

## Global / File-Static State
None.

## Key Functions / Methods

### ReadWeightConfig
- **Signature:** `weightconfig_t *ReadWeightConfig(char *filename)`
- **Purpose:** Parses a weight config file from disk and returns an allocated `weightconfig_t`
- **Inputs:** Path to the weight config script file
- **Outputs/Return:** Pointer to heap-allocated `weightconfig_t`; presumably `NULL` on failure
- **Side effects:** Heap allocation; file I/O via botlib script parser
- **Calls:** Defined in `be_ai_weight.c`
- **Notes:** Result must be freed with `FreeWeightConfig`

### FreeWeightConfig
- **Signature:** `void FreeWeightConfig(weightconfig_t *config)`
- **Purpose:** Releases all memory associated with a weight configuration, including the fuzzy separator trees
- **Inputs:** Pointer to a previously allocated `weightconfig_t`
- **Outputs/Return:** void
- **Side effects:** Heap deallocation
- **Calls:** Defined in `be_ai_weight.c`

### WriteWeightConfig
- **Signature:** `qboolean WriteWeightConfig(char *filename, weightconfig_t *config)`
- **Purpose:** Serializes a weight configuration back to a file
- **Inputs:** Output filename, source config
- **Outputs/Return:** `qtrue` on success, `qfalse` on failure
- **Side effects:** File I/O (write)

### FindFuzzyWeight
- **Signature:** `int FindFuzzyWeight(weightconfig_t *wc, char *name)`
- **Purpose:** Looks up a weight index by name within a config
- **Inputs:** Config pointer, weight name string
- **Outputs/Return:** Integer index into `wc->weights[]`, or `-1`/sentinel on not found

### FuzzyWeight
- **Signature:** `float FuzzyWeight(int *inventory, weightconfig_t *wc, int weightnum)`
- **Purpose:** Evaluates a named weight against the bot's current inventory array using fuzzy interval logic
- **Inputs:** Bot inventory array (indexed by item type), config, weight index
- **Outputs/Return:** Scalar float weight value for use in goal/decision scoring
- **Side effects:** None (pure evaluation)

### FuzzyWeightUndecided
- **Signature:** `float FuzzyWeightUndecided(int *inventory, weightconfig_t *wc, int weightnum)`
- **Purpose:** Variant of `FuzzyWeight` that returns a value when the fuzzy evaluation falls into an undecided/boundary region
- **Notes:** Behavior differs from `FuzzyWeight` at separator boundaries; exact semantics defined in `be_ai_weight.c`

### EvolveWeightConfig / InterbreedWeightConfigs
- **Signature:** `void EvolveWeightConfig(weightconfig_t *config)` / `void InterbreedWeightConfigs(weightconfig_t *config1, weightconfig_t *config2, weightconfig_t *configout)`
- **Purpose:** Genetic-algorithm-style mutation and crossover of weight configs for offline bot tuning
- **Side effects:** Mutates `config` in-place or writes blended result into `configout`

### BotShutdownWeights
- **Signature:** `void BotShutdownWeights(void)`
- **Purpose:** Flushes any internally cached weight configurations on shutdown
- **Side effects:** Heap deallocation of cached state; called during botlib teardown

## Control Flow Notes
This header is consumed primarily by `be_ai_goal.c` and `be_ai_move.c`, which call `FindFuzzyWeight` at init time and `FuzzyWeight`/`FuzzyWeightUndecided` per-frame during goal evaluation. `ReadWeightConfig` is called during bot initialization; `BotShutdownWeights` is called at shutdown. Evolution/interbreeding functions are offline/tooling utilities, not part of the normal game loop.

## External Dependencies
- `MAX_QPATH` — defined in `q_shared.h` (engine shared header)
- `qboolean` — engine boolean typedef from `q_shared.h`
- `WT_BALANCE` (`1`) — constant used to tag separator nodes of balance type; consumed by `be_ai_weight.c`
- `MAX_WEIGHTS` (`128`) — caps the static weight array in `weightconfig_t`
- Implementation: `be_ai_weight.c`
