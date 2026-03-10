# code/game/be_ai_weap.h — Enhanced Analysis

## Architectural Role

This header bridges the game VM (`code/game/`) with the botlib weapon AI subsystem (`code/botlib/be_ai_weap.c`). It defines the public contract for bot weapon selection and ballistics reasoning, allowing per-bot weapon state management and data-driven weapon evaluation via external weight files. Bots query this API per-frame to make firing and weapon-switch decisions, making it a critical component of bot combat behavior.

## Key Cross-References

### Incoming (who depends on this file)
- **code/game/g_bot.c** — Bot lifecycle management; allocates/frees weapon states during bot spawn/death via `BotAllocWeaponState` / `BotFreeWeaponState`
- **code/game/ai_dmq3.c** / **code/game/ai_dmnet.c** — Bot AI FSM nodes; call `BotChooseBestFightWeapon` per-frame during combat decision-making, and `BotGetWeaponInfo` to query projectile properties for arc prediction
- **code/game/be_ai_*.h** (other botlib headers in game tree) — Cross-include for type definitions (`weaponinfo_t`, `projectileinfo_t`) used throughout bot AI logic

### Outgoing (what this file depends on)
- **code/botlib/be_ai_weap.c** — Implementation of all declared functions; maintains per-state weapon weight tables loaded from `.cfg` files
- **botlib.h** (implicit via `MAX_STRINGFIELD`) — Shared botlib string field size constant
- **q_shared.h** — Vector math type `vec3_t` for recoil and offset data
- **code/server/sv_bot.c** — Supplies weapon weight filenames and bot state handles through `trap_BotLib*` syscall dispatch

## Design Patterns & Rationale

**Handle-Based State Management**: The weapon state is opaque (identified by an integer handle), not directly exposed. This:
- Isolates botlib's internal state representation from game code
- Allows multiple concurrent weapon states (one per bot)
- Enables hot-swappable botlib implementations without recompiling game

**Data-Driven Weapon Selection**: `BotLoadWeaponWeights` loads a configuration file, suggesting the weapon choice is a weighted fuzzy-logic evaluation (not hard-coded priorities). This allows:
- Map-specific weapon balance tuning
- Gameplay rebalancing without code recompilation
- Different AI profiles (aggressive vs. defensive) via different weight files

**Embedded Projectile Info**: `weaponinfo_t` embeds a full `projectileinfo_t` struct rather than a pointer. Rationale:
- Most weapons use their projectile frequently; avoiding indirection improves cache locality
- Simplifies memory management (no separate alloc for projectile data)
- Projectiles are immutable at runtime

## Data Flow Through This File

```
[Map Load]
  ↓
[Server: BotSetupWeaponAI() — one-time init]
  ↓
[Per-Bot Spawn]
  → BotAllocWeaponState() → returns weaponstate handle
  → BotLoadWeaponWeights(weaponstate, "weapons.cfg") → populates weight tables
  ↓
[Per-Frame Combat]
  → inventory (ammo counts) from game state
  → BotChooseBestFightWeapon(weaponstate, *inventory) → weapon number
  → BotGetWeaponInfo(weaponstate, weapon, *weaponinfo) → fills struct
  → AI uses weaponinfo.proj.gravity, .speed, .recoil to plan movement/aim
  ↓
[Per-Bot Death]
  → BotFreeWeaponState(weaponstate) → release resources
  ↓
[Server Shutdown]
  → BotShutdownWeaponAI() — cleanup
```

## Learning Notes

**Quake III-Era Design Patterns**:
- Integer handles (not pointers) for opaque resources — common for DLL/module boundaries to avoid ABI brittleness
- Data-driven configuration via external `.cfg` files — gameplay tuning without recompilation
- Embedding full structs (vs. pointers) when data is small and frequently accessed — memory layout matters on 2000s hardware

**Connections to Game Engine Concepts**:
- **Resource Pooling**: Weapon states are likely pre-allocated in a fixed-size pool, reused across bot lifetimes (no malloc per frame)
- **Fuzzy Logic**: The weight file likely encodes fuzzy membership functions (e.g., "how good is rocket launcher given X ammo vs. Y health?")
- **Deterministic AI**: Weapon selection is fully determined by inventory state + loaded weights, enabling replayed demos and deterministic netplay

**Notable Absence**:
- No per-weapon cooldown/ammo tracking here; that's managed separately in the game VM (`botlib_import_t` provides ammo state read-only)
- No firing commands generated here; `be_ea.c` (EA layer) does that

## Potential Issues

- **Integer Handle Type Safety**: `int weaponstate` offers no compile-time validation; a bad handle passed to `BotGetWeaponInfo` will silently corrupt or crash. Modern engines use opaque structs or generational indices.
- **No Handle Validation**: No bounds checking visible in this header; assumes game code never passes freed or out-of-range handles.
- **Mutable Weapon Info**: `weaponinfo_t` returned by `BotGetWeaponInfo` is written to by botlib; caller must ensure no aliasing issues if storing across frames (though this is likely fine by design).
