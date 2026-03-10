# code/botlib/be_ai_weap.c — Enhanced Analysis

## Architectural Role

This file implements the weapon-selection layer within the botlib subsystem, bridging the game VM's bot AI pipeline to fuzzy-logic weapon scoring. It loads and manages shared weapon configuration data (names, projectile properties, ballistics) and per-bot weapon-weight tuning, enabling bots to dynamically select the best weapon during combat. The file exemplifies botlib's layered architecture: game VM calls `trap_BotLib*` syscalls (exposed via `be_interface.c`) which dispatch into botlib modules like this one, maintaining clean separation and allowing botlib to be recompiled independently (as in `bspc/`, the offline map compiler).

## Key Cross-References

### Incoming (who depends on this file)
- **code/game/** (via `trap_BotLoadWeaponWeights`, `trap_BotChooseBestFightWeapon`, `trap_BotGetWeaponInfo`): game VM's `g_bot.c` and AI FSM files (`ai_dmq3.c`, `ai_dmnet.c`) call these to load per-bot weapon weights and select weapons during combat
- **code/botlib/be_interface.c**: exports the public `botlib_export_t` vtable; all game VM calls funnel through syscall dispatch here
- **code/botlib/** initialization layer: `BotSetupWeaponAI` / `BotShutdownWeaponAI` are called during botlib startup/shutdown, likely from `be_interface.c`

### Outgoing (what this file depends on)
- **code/botlib/be_ai_weight.h**: `ReadWeightConfig`, `FreeWeightConfig`, `FindFuzzyWeight`, `FuzzyWeight` (fuzzy logic scoring)
- **code/botlib/l_script.h, l_precomp.h, l_struct.h**: `LoadSourceFile`, `PC_ReadToken`, `ReadStructure`, `WriteStructure` (config file parsing via reflection)
- **code/botlib/l_libvar.h**: `LibVarValue`, `LibVarString`, `LibVarSet` (libvar cvar-like system)
- **code/botlib/l_memory.h**: `GetClearedMemory`, `GetClearedHunkMemory`, `FreeMemory` (memory pools)
- **code/botlib/be_interface.h**: `botimport` vtable (Print, file access, error handling)
- **code/game/be_ai_weap.h**: public header with `weaponinfo_t`, `projectileinfo_t` type definitions (defines the API contract with game VM)

## Design Patterns & Rationale

**1. Reflection-based Binary Structure Serialization**
The file uses `fielddef_t` arrays (`weaponinfo_fields[]`, `projectileinfo_fields[]`) to describe struct layout, paired with generic `ReadStructure`/`WriteStructure` functions. This avoids hand-written parser code and allows tight coupling between source (script file) and in-memory representation. It's a lightweight reflection system idiomatic to the Quake engine toolchain.

**2. Per-Entity State Pools with Handle Indirection**
`BotAllocWeaponState` / `BotFreeWeaponState` manage a pre-allocated array `botweaponstates[1..MAX_CLIENTS]`, indexed by handle. `BotWeaponStateFromHandle` validates and retrieves state. This pattern is used throughout botlib (`be_ai_move.c`, `be_ai_goal.c`) and game VM (`g_client.c`) for cache-friendly, garbage-free entity state management — a direct consequence of Quake's predictable frame loop and no garbage collection.

**3. Separation of Global Config from Per-Entity Tuning**
- Global `weaponconfig` (loaded once) contains static weapon/projectile data shared by all bots
- Per-bot `bot_weaponstate_t` holds only pointers to the weight config and weight-index array

This minimizes per-bot memory and enables efficient reconfiguration of AI behavior (reload weights) without touching the weapon definitions.

**4. Lazy Weapon Weight Loading**
Unlike static game configs, fuzzy weight configs are loaded on demand via `BotLoadWeaponWeights(handle, filename)`. This defers cost until a bot is actually initialized, scaling to many idle bot slots without memory penalty.

**5. Subsystem Boundary Isolation via Import Vtable**
The file never calls engine functions directly; all communication is through `botimport.*`. This enables botlib to be compiled against different backends (runtime engine vs. offline `bspc` compiler), achieving code reuse without link-time coupling.

## Data Flow Through This File

**Initialization Phase (once at botlib startup):**
1. Engine calls `BotSetupWeaponAI()` or via syscall dispatch
2. Read libvar `weaponconfig` (default: weapon config file path)
3. Call `LoadWeaponConfig(filename)`:
   - Open and parse script file via `LoadSourceFile` + `PC_ReadToken`
   - For each `weaponinfo` block: parse structure via reflection, store in `wc->weaponinfo[weaponnum]`
   - For each `projectileinfo` block: parse structure, store sequentially in `wc->projectileinfo[]`
   - Post-parse fix-up: for each weapon, find its projectile by name and copy `projectileinfo_t` into weapon's `proj` field
   - Return allocated `weaponconfig`
4. Store global `weaponconfig` pointer

**Per-Bot Initialization (on bot spawn):**
1. Game VM allocates bot state via `BotAllocWeaponState()` → returns handle (1..MAX_CLIENTS)
2. Game VM calls `BotLoadWeaponWeights(handle, weightfile)`:
   - Retrieve `bot_weaponstate_t*` from handle
   - Call `ReadWeightConfig(weightfile)` to load fuzzy weight definitions
   - Call `WeaponWeightIndex(weightconfig, weaponconfig)`:
     - For each weapon in weaponconfig, call `FindFuzzyWeight(weightconfig, weapon.name)` to map weapon name → weight index
     - Return allocated `int[]` array
   - Store weight config and weight-index array in `botweaponstates[handle]`

**Per-Frame Weapon Selection (combat loop):**
1. Game VM calls `BotChooseBestFightWeapon(handle, inventory[])` (inventory = ammo counts, health, powerups, etc.)
2. Loop through all weapons in `weaponconfig->weaponinfo[0..numweapons]`:
   - Skip invalid weapons
   - Get weight index via `ws->weaponweightindex[i]`
   - Score weapon: `FuzzyWeight(inventory, weightconfig, index)` (returns float score based on fuzzy membership functions)
   - Track highest-scoring weapon
3. Return best weapon number (or 0 if no valid weapon scores > 0)

**Shutdown Phase:**
1. `BotShutdownWeaponAI()`:
   - Call `BotFreeWeaponState(handle)` for each allocated bot
   - Free global `weaponconfig`

## Learning Notes

**Idiomatic Patterns of the Quake Engine (circa 1999):**

1. **No Object-Oriented Abstraction**: Everything is procedural C with handle-based indirection (`BotWeaponStateFromHandle`). Today's engines use ECS, OOP class hierarchies, or scripting VMs; Quake uses flat arrays and handles. This is simpler but less flexible.

2. **Lightweight Reflection without Meta-Programming**: The `fielddef_t` array pattern is a manual equivalent of C# reflection or Java annotations. It's self-contained (no external dependency on serialization libraries) and trivial to debug.

3. **Fuzzy Logic for AI**: Weapon selection isn't hard-coded ("if ammo > 100 use shotgun"). Instead, each weapon has a fuzzy weight function defined in a text file, evaluated against the bot's inventory state. This was cutting-edge for 1999; modern engines use neural networks or behavior trees, but fuzzy logic remains effective for rule-based AI.

4. **Global Singleton Config Pattern**: One global `weaponconfig` shared by all bots. No dynamic reloading or per-server customization at runtime (config is static per map). This works because Quake serves one map per server instance, and all game state is reset on map load.

5. **Strict Subsystem Isolation via Function Pointers**: `botlib_import_t botimport` is a vtable that allows the entire botlib module to be decoupled from the engine. The same botlib code runs in the runtime server and in the offline `bspc` tool, with different import implementations. This was essential before dynamic linking became mainstream.

**Conceptual Connection to Modern Engine Architecture:**

- **Weapon Config ↔ Scriptable Item Definitions**: Modern engines use YAML/JSON specs with reflection hydration; Quake uses fielddef tables. Both achieve the same goal: data-driven weapon properties.
- **Fuzzy Weight System ↔ AI Decision Systems**: Modern engines might use decision trees, utility scoring, or neural networks. Quake's fuzzy weights are a simpler, interpretable predecessor to utility-based AI.
- **Per-Entity State Pools ↔ ECS**: Quake's handle-based entity state (`botweaponstates[]`) is a precursor to ECS component storage. The memory layout is dense and cache-friendly, which ECS also targets.

## Potential Issues

1. **Silent Failure Mode in `BotChooseBestFightWeapon`**: If the weight config doesn't define weights for any weapon, all scores will be 0, and the function returns 0 (no weapon). The game VM must handle this case (defensive coding). No error is logged. Consider adding a warning if all scores are 0.

2. **`BotResetWeaponState` is Nearly a No-Op**: The function saves and restores weight pointers but doesn't actually reset any state. The commented line `// Com_Memset(ws, 0, sizeof(bot_weaponstate_t));` suggests it was intended to clear state while preserving config. As written, it's a vestigial stub. Either document its purpose or remove it.

3. **Linear Projectile Name Lookup in `LoadWeaponConfig`**: For each weapon, a linear search finds the matching projectile (line ~310). With typical counts (32 items), this is O(n²) = acceptable. If you ever scale to thousands of weapons, add a hash table.

4. **No Reload Mechanism**: Once `weaponconfig` is loaded, there's no `ReloadWeaponConfig`. If you change weapon data mid-game, you must restart. For a multiplayer server, this is fine (config is per-map), but it limits live iteration during development. Consider adding a reload command for debugging.

5. **Hardcoded MAX_CLIENTS Assumption**: The array `botweaponstates[MAX_CLIENTS+1]` is pre-allocated at subsystem init with a fixed size. If `MAX_CLIENTS` changes (recompilation required), bot state must be reallocated. Not a functional issue but inflexible.
