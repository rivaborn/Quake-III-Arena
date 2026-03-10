# code/botlib/be_ai_weight.c

## File Purpose
Implements a fuzzy logic weight evaluation system for the Q3 bot AI, parsing hierarchical weight configuration files and evaluating weighted decisions based on bot inventory state. It supports both deterministic and randomized ("undecided") weight lookups, as well as genetic-algorithm-style evolution and interbreeding of weight configs.

## Core Responsibilities
- Parse `weight` config files into `weightconfig_t` trees of `fuzzyseperator_t` nodes
- Cache loaded weight configs in a global file list (`weightFileList`) to avoid redundant disk reads
- Evaluate fuzzy weights given a bot's inventory array (deterministic and stochastic variants)
- Support evolutionary mutation (`EvolveWeightConfig`) and blending (`InterbreedWeightConfigs`) of weight configs
- Free and shut down weight config memory on demand

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `weightconfig_t` | struct (defined in `be_ai_weight.h`) | Top-level config holding an array of named `weight_t` entries |
| `weight_t` | struct (defined in `be_ai_weight.h`) | Single named weight with a linked tree of `fuzzyseperator_t` |
| `fuzzyseperator_t` | struct (defined in `be_ai_weight.h`) | Node in a switch/case decision tree; holds index, value, weight, min/maxweight, child, next |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `weightFileList` | `weightconfig_t *[128]` | file-static (global linkage) | Cache of loaded weight configs, indexed by slot; avoids re-parsing files when `bot_reloadcharacters` is off |

## Key Functions / Methods

### ReadValue
- **Signature:** `int ReadValue(source_t *source, float *value)`
- **Purpose:** Reads a single numeric token from a script source into `*value`; treats leading `-` as zero with a warning.
- **Inputs:** Script parser source, pointer to float output
- **Outputs/Return:** `qtrue` on success, `qfalse` on parse error
- **Side effects:** Advances script parse position; emits source warning/error
- **Calls:** `PC_ExpectAnyToken`, `PC_ExpectTokenType`, `SourceWarning`, `SourceError`

### ReadFuzzyWeight
- **Signature:** `int ReadFuzzyWeight(source_t *source, fuzzyseperator_t *fs)`
- **Purpose:** Reads a `return` statement into a `fuzzyseperator_t`; handles optional `balance(w, min, max)` syntax or plain scalar.
- **Inputs:** Source, pointer to node to populate
- **Outputs/Return:** `qtrue`/`qfalse`
- **Side effects:** Modifies `fs->type`, `fs->weight`, `fs->minweight`, `fs->maxweight`
- **Calls:** `PC_CheckTokenString`, `PC_ExpectTokenString`, `ReadValue`

### ReadFuzzySeperators_r
- **Signature:** `fuzzyseperator_t *ReadFuzzySeperators_r(source_t *source)`
- **Purpose:** Recursively parses a `switch(index) { case N: ... default: ... }` block into a linked list of `fuzzyseperator_t` nodes with optional children.
- **Inputs:** Script source
- **Outputs/Return:** Head of parsed node list, or `NULL` on error
- **Side effects:** Heap allocation via `GetClearedMemory`; frees partial results on error via `FreeFuzzySeperators_r`
- **Calls:** `PC_ExpectTokenString`, `PC_ExpectTokenType`, `PC_ExpectAnyToken`, `ReadFuzzyWeight`, recursive self-call, `FreeFuzzySeperators_r`, `SourceError`, `SourceWarning`
- **Notes:** Adds a synthetic default node (weight=0) if none is declared.

### ReadWeightConfig
- **Signature:** `weightconfig_t *ReadWeightConfig(char *filename)`
- **Purpose:** Loads and parses a `.c`-style weight config file; returns cached copy if `bot_reloadcharacters` is off and file was previously loaded.
- **Inputs:** Filename string
- **Outputs/Return:** Pointer to allocated `weightconfig_t`, or `NULL` on failure
- **Side effects:** Allocates heap memory; stores result in `weightFileList`; prints load messages via `botimport.Print`
- **Calls:** `LibVarGetValue`, `PC_SetBaseFolder`, `LoadSourceFile`, `GetClearedMemory`, `PC_ReadToken`, `ReadFuzzySeperators_r`, `ReadFuzzyWeight`, `FreeWeightConfig`, `FreeSource`, `botimport.Print`

### FuzzyWeight_r / FuzzyWeight
- **Signature:** `float FuzzyWeight_r(int *inventory, fuzzyseperator_t *fs)` / `float FuzzyWeight(int *inventory, weightconfig_t *wc, int weightnum)`
- **Purpose:** Evaluates a deterministic fuzzy weight by traversing the separator tree, interpolating linearly between adjacent case values.
- **Inputs:** Bot inventory array, separator node or config+index
- **Outputs/Return:** Interpolated float weight
- **Side effects:** None
- **Calls:** Recursive self-call

### FuzzyWeightUndecided_r / FuzzyWeightUndecided
- **Signature:** Same pattern as above
- **Purpose:** Like `FuzzyWeight_r` but uses `random()` to sample within `[minweight, maxweight]` for stochastic behavior.
- **Side effects:** Uses `random()` (global RNG state)

### EvolveWeightConfig
- **Signature:** `void EvolveWeightConfig(weightconfig_t *config)`
- **Purpose:** Mutates all `WT_BALANCE` leaf nodes via Gaussian-like random perturbation; 1% chance of full-range leap vs. 50% scale step.
- **Calls:** `EvolveFuzzySeperator_r`, `random()`, `crandom()`

### InterbreedWeightConfigs
- **Signature:** `void InterbreedWeightConfigs(weightconfig_t *config1, weightconfig_t *config2, weightconfig_t *configout)`
- **Purpose:** Averages `WT_BALANCE` weights from two configs into a third; validates structural compatibility.
- **Side effects:** Modifies `configout`; prints errors via `botimport.Print` on mismatch

### BotShutdownWeights
- **Signature:** `void BotShutdownWeights(void)`
- **Purpose:** Frees all cached weight configs in `weightFileList` and NULLs their slots.
- **Side effects:** Frees heap memory; clears `weightFileList`

## Control Flow Notes
This file is not part of the frame/render loop. It is invoked during **bot initialization** (`ReadWeightConfig`) and **shutdown** (`BotShutdownWeights`). `FuzzyWeight` / `FuzzyWeightUndecided` are called per-decision during the bot AI update cycle from the goal/move/weapon layers. `EvolveWeightConfig` and `InterbreedWeightConfigs` are called externally for bot difficulty tuning.

## External Dependencies
- `l_precomp.h` / `l_script.h`: `source_t`, `token_t`, `PC_*` parsing functions, `LoadSourceFile`, `FreeSource`
- `l_memory.h`: `GetClearedMemory`, `FreeMemory`
- `l_libvar.h`: `LibVarGetValue` (reads `bot_reloadcharacters`)
- `be_interface.h`: `botimport` (print/error callbacks)
- `be_ai_weight.h`: Type definitions for `weightconfig_t`, `weight_t`, `fuzzyseperator_t`, `WT_BALANCE`, `MAX_WEIGHTS`
- `q_shared.h`: `qboolean`, `Q_strncpyz`, `random()`, `crandom()`
- `Sys_MilliSeconds` (DEBUG only): defined in platform layer
