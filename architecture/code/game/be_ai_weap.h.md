# code/game/be_ai_weap.h

## File Purpose
Public header defining data structures and function prototypes for the bot weapon AI subsystem. It describes projectile and weapon properties used by the botlib to reason about weapon selection and ballistics.

## Core Responsibilities
- Define flags for projectile behavior (window damage, return-to-owner)
- Define flags for weapon firing behavior (key-up fire release)
- Define damage type bitmasks (impact, radial, visible)
- Declare `projectileinfo_t` and `weaponinfo_t` structs used throughout the bot weapon system
- Expose the weapon AI lifecycle API (setup, shutdown, alloc, free, reset)
- Expose weapon selection and information query functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `projectileinfo_t` | struct | Describes a projectile: physics (gravity, bounce, push), damage (amount, radius, type), and behavioral flags |
| `weaponinfo_t` | struct | Describes a weapon: spread, speed, recoil, ammo, timing parameters, level, and an embedded `projectileinfo_t` for its projectile |

## Global / File-Static State
None.

## Key Functions / Methods

### BotSetupWeaponAI
- Signature: `int BotSetupWeaponAI(void)`
- Purpose: Initializes the weapon AI subsystem.
- Inputs: None
- Outputs/Return: Integer status code (success/failure convention).
- Side effects: Allocates internal weapon AI state; defined elsewhere.
- Calls: Not inferable from this file.
- Notes: Must be called before any other weapon AI function.

### BotShutdownWeaponAI
- Signature: `void BotShutdownWeaponAI(void)`
- Purpose: Tears down the weapon AI subsystem and releases resources.
- Inputs: None
- Outputs/Return: None
- Side effects: Frees internal state; defined elsewhere.
- Calls: Not inferable from this file.

### BotChooseBestFightWeapon
- Signature: `int BotChooseBestFightWeapon(int weaponstate, int *inventory)`
- Purpose: Selects the optimal weapon for combat given the bot's current inventory and loaded weapon weights.
- Inputs: `weaponstate` — handle to bot's weapon state; `inventory` — array of item counts indexed by ammo/weapon index.
- Outputs/Return: Integer weapon number of the best weapon to use.
- Side effects: None (query only); defined elsewhere.
- Calls: Not inferable from this file.
- Notes: Depends on weights loaded via `BotLoadWeaponWeights`.

### BotGetWeaponInfo
- Signature: `void BotGetWeaponInfo(int weaponstate, int weapon, weaponinfo_t *weaponinfo)`
- Purpose: Fills a `weaponinfo_t` struct with data for the specified weapon number.
- Inputs: `weaponstate` — weapon state handle; `weapon` — weapon number; `weaponinfo` — output buffer.
- Outputs/Return: Populated `*weaponinfo` (out parameter).
- Side effects: Writes to caller-provided struct; defined elsewhere.

### BotLoadWeaponWeights
- Signature: `int BotLoadWeaponWeights(int weaponstate, char *filename)`
- Purpose: Loads a weapon weight configuration file into the given weapon state for use by `BotChooseBestFightWeapon`.
- Inputs: `weaponstate` — weapon state handle; `filename` — path to weights file.
- Outputs/Return: Integer success/failure code.
- Side effects: Reads from filesystem; populates internal weight tables; defined elsewhere.

### BotAllocWeaponState / BotFreeWeaponState / BotResetWeaponState
- **Alloc**: Allocates and returns a new weapon state handle.
- **Free**: Releases resources associated with a weapon state handle.
- **Reset**: Clears all state for a weapon state handle without freeing it.

## Control Flow Notes
This header is consumed both by the botlib implementation (`code/botlib/be_ai_weap.c`) and the game-side bot logic (`code/game/`). During bot initialization, `BotSetupWeaponAI` is called once; per-bot setup calls `BotAllocWeaponState` and `BotLoadWeaponWeights`. Per-frame, `BotChooseBestFightWeapon` and `BotGetWeaponInfo` are used to drive weapon selection decisions. Shutdown calls `BotFreeWeaponState` then `BotShutdownWeaponAI`.

## External Dependencies
- `MAX_STRINGFIELD` — defined in botlib shared headers (e.g., `botlib.h` or `be_aas.h`)
- `vec3_t` — defined in `q_shared.h`
- All function bodies defined in `code/botlib/be_ai_weap.c`
