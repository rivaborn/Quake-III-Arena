# code/botlib/be_ai_weap.c

## File Purpose
Implements the weapon AI subsystem for Q3 bots, responsible for loading weapon/projectile configuration data from script files, managing per-bot weapon state, and selecting the best weapon to use in combat via fuzzy-weight evaluation.

## Core Responsibilities
- Load and parse weapon configuration files (`weapons.c`) into `weaponconfig_t` structures
- Load per-bot fuzzy weight configurations for weapon selection scoring
- Map parsed weapon names to fuzzy weight indices via `WeaponWeightIndex`
- Allocate and free per-bot `bot_weaponstate_t` handles (one per client slot)
- Evaluate all valid weapons against a bot's inventory using fuzzy logic to select the best fight weapon
- Provide weapon info lookup by weapon number for external callers
- Initialize and shut down the global weapon AI subsystem

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `weaponconfig_t` | struct | Global weapon/projectile database loaded from file; holds arrays of `weaponinfo_t` and `projectileinfo_t` |
| `bot_weaponstate_t` | struct | Per-bot state: holds pointer to fuzzy weight config and weapon-to-weight index array |
| `weaponinfo_fields[]` | static `fielddef_t[]` | Reflection table mapping weapon script field names to `weaponinfo_t` member offsets |
| `projectileinfo_fields[]` | static `fielddef_t[]` | Reflection table mapping projectile script field names to `projectileinfo_t` member offsets |
| `weaponinfo_struct` | static `structdef_t` | Struct descriptor for generic structure reader (size + field table) |
| `projectileinfo_struct` | static `structdef_t` | Struct descriptor for projectile structure reader |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `botweaponstates` | `bot_weaponstate_t *[MAX_CLIENTS+1]` | static | Per-client weapon state handles, indexed 1..MAX_CLIENTS |
| `weaponconfig` | `weaponconfig_t *` | static | Singleton global weapon/projectile database shared across all bots |

## Key Functions / Methods

### LoadWeaponConfig
- **Signature:** `weaponconfig_t *LoadWeaponConfig(char *filename)`
- **Purpose:** Parses a botlib script file containing `weaponinfo` and `projectileinfo` blocks into an allocated `weaponconfig_t`. Links each weapon's projectile by name after parsing.
- **Inputs:** `filename` — path to the weapon config script
- **Outputs/Return:** Pointer to allocated `weaponconfig_t`, or `NULL` on any error
- **Side effects:** Allocates hunk memory; calls `botimport.Print` on errors/success; calls `LoadSourceFile`/`FreeSource`
- **Calls:** `LibVarValue`, `LibVarSet`, `PC_SetBaseFolder`, `LoadSourceFile`, `GetClearedHunkMemory`, `PC_ReadToken`, `ReadStructure`, `FreeMemory`, `FreeSource`, `Com_Memset`, `Com_Memcpy`, `botimport.Print`
- **Notes:** Weapons are indexed by their `number` field, not sequentially. Projectile linkage is done in a post-parse fix-up loop. Returns `NULL` and frees `wc` on any validation failure.

### WeaponWeightIndex
- **Signature:** `int *WeaponWeightIndex(weightconfig_t *wwc, weaponconfig_t *wc)`
- **Purpose:** Builds an array mapping each weapon slot index to its corresponding fuzzy weight index within `wwc`.
- **Inputs:** `wwc` — loaded weight config; `wc` — weapon config
- **Outputs/Return:** Heap-allocated `int[]` of size `wc->numweapons`
- **Side effects:** Allocates cleared memory
- **Calls:** `GetClearedMemory`, `FindFuzzyWeight`

### BotChooseBestFightWeapon
- **Signature:** `int BotChooseBestFightWeapon(int weaponstate, int *inventory)`
- **Purpose:** Scores all valid weapons using fuzzy logic against the bot's current inventory and returns the index of the highest-scoring weapon.
- **Inputs:** `weaponstate` — bot handle; `inventory` — bot's current item/ammo counts
- **Outputs/Return:** Best weapon index (into `weaponconfig->weaponinfo[]`), or `0` if none
- **Side effects:** None (read-only query)
- **Calls:** `BotWeaponStateFromHandle`, `FuzzyWeight`
- **Notes:** Returns `0` (no weapon) if weight config is absent, weaponconfig is null, or no valid weapon scores above 0.

### BotLoadWeaponWeights
- **Signature:** `int BotLoadWeaponWeights(int weaponstate, char *filename)`
- **Purpose:** Loads a fuzzy weight file for a bot's weapon selection and builds the weapon-to-weight index.
- **Inputs:** `weaponstate` — bot handle; `filename` — weight config file path
- **Outputs/Return:** `BLERR_NOERROR` on success, `BLERR_CANNOTLOADWEAPONWEIGHTS` / `BLERR_CANNOTLOADWEAPONCONFIG` on failure
- **Side effects:** Frees any previously loaded weights; allocates new weight config and index
- **Calls:** `BotWeaponStateFromHandle`, `BotFreeWeaponWeights`, `ReadWeightConfig`, `WeaponWeightIndex`, `botimport.Print`

### BotSetupWeaponAI
- **Signature:** `int BotSetupWeaponAI(void)`
- **Purpose:** Subsystem initialization — reads the `weaponconfig` libvar and loads the global weapon config.
- **Inputs:** None
- **Outputs/Return:** `BLERR_NOERROR` or `BLERR_CANNOTLOADWEAPONCONFIG`
- **Side effects:** Sets global `weaponconfig`; calls `LoadWeaponConfig`
- **Calls:** `LibVarString`, `LoadWeaponConfig`, `botimport.Print`, optionally `DumpWeaponConfig`

### BotShutdownWeaponAI
- **Signature:** `void BotShutdownWeaponAI(void)`
- **Purpose:** Frees the global weapon config and all active bot weapon states.
- **Side effects:** Nulls `weaponconfig`; calls `BotFreeWeaponState` for each active slot
- **Calls:** `FreeMemory`, `BotFreeWeaponState`

- **Notes on trivial helpers:** `BotValidWeaponNumber` bounds-checks a weapon index. `BotWeaponStateFromHandle` validates and returns a state pointer. `BotAllocWeaponState`/`BotFreeWeaponState` manage the `botweaponstates[]` array. `BotResetWeaponState` is a no-op stub preserving weight pointers. `BotGetWeaponInfo` copies a `weaponinfo_t` by value to the caller.

## Control Flow Notes
- **Init:** `BotSetupWeaponAI` is called once at botlib startup, loading the shared `weaponconfig`.
- **Per-bot init:** `BotAllocWeaponState` → `BotLoadWeaponWeights` sets up each bot's scoring state.
- **Per-frame:** `BotChooseBestFightWeapon` is called by higher-level bot AI when evaluating combat decisions.
- **Shutdown:** `BotShutdownWeaponAI` tears down all state; individual bots call `BotFreeWeaponState`.

## External Dependencies
- `l_script.h` / `l_precomp.h` — `LoadSourceFile`, `PC_ReadToken`, `FreeSource`, `PC_SetBaseFolder`
- `l_struct.h` — `ReadStructure`, `WriteStructure`, `fielddef_t`, `structdef_t`
- `be_ai_weight.h` — `weightconfig_t`, `ReadWeightConfig`, `FreeWeightConfig`, `FindFuzzyWeight`, `FuzzyWeight`
- `be_interface.h` — `botimport` (global import table providing `Print`)
- `l_libvar.h` — `LibVarValue`, `LibVarString`, `LibVarSet`
- `l_memory.h` — `GetClearedMemory`, `GetClearedHunkMemory`, `FreeMemory`
- `../game/be_ai_weap.h` — `weaponinfo_t`, `projectileinfo_t` type definitions (defined elsewhere)
- `botlib.h` — `BLERR_*` error codes, `MAX_CLIENTS`
