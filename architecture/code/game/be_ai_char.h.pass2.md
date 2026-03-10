# code/game/be_ai_char.h — Enhanced Analysis

## Architectural Role

This header exposes botlib's bot character personality and skill system as a cross-layer interface. It bridges the game VM (which spawns bots and needs their behavioral traits) to botlib's internal character database. Though physically located in `code/game/`, the canonical implementation lives in `code/botlib/be_ai_char.c` and this is part of botlib's public `botlib_export_t` vtable—making it a gateway between the server's bot lifecycle management (`sv_bot.c`) and per-frame AI decision-making in the game module (`ai_*.c`).

## Key Cross-References

### Incoming (who depends on this file)
- **code/server/sv_bot.c** — Bot initialization and teardown; calls `BotLoadCharacter` at bot spawn and `BotFreeCharacter` at removal
- **code/game/g_bot.c** — High-level bot entity spawning in the game VM; invokes character load via syscall or direct link
- **code/game/ai_dmq3.c, ai_dmnet.c, ai_*.c** — Per-frame AI FSM execution; calls `Characteristic_*` accessors to retrieve personality traits (aggression, chat tendency, weapon preference biases, etc.)
- **code/botlib/be_interface.c** — Vtable export layer; wraps these functions for `botlib_export_t` delivery to the server

### Outgoing (what this file depends on)
- **code/botlib/be_ai_char.c** — Implementation; loads `.cfg` character files, parses characteristics by index, caches loaded profiles
- **code/botlib/l_memory.c, l_script.c, l_struct.c** — Botlib's internal utilities for memory, config parsing, and structured data handling
- **code/qcommon/files.c** — Virtual filesystem for `.cfg` character file access (indirectly through botlib's import layer)

## Design Patterns & Rationale

**Opaque Handle Pattern**: Characters referenced by `int` IDs (not struct pointers) enforces strict encapsulation—botlib controls memory layout and lifetime entirely. This avoids VM/engine ABI breaks if the internal `bot_character_t` structure changes.

**Bounded Accessors (`_B` suffix)**: The `Characteristic_BFloat` and `Characteristic_BInteger` variants clamp results without caller-side branches, a convenience common in 1990s–2000s game code. Encodes early validation strategy (bounds-check at retrieval rather than on parse).

**Skill-Parameterized Loading**: The `float skill` argument to `BotLoadCharacter` enables dynamic difficulty scaling—botlib blends characteristic values (e.g., reaction time, aim precision) based on skill level at load time, supporting bot difficulty tiers without duplication of character files.

**Type-Specific Accessors Over Generics**: Rather than a single `GetCharacteristic(handle, index, &outValue)` with implicit type casting, the Q3A design uses `Characteristic_Float`, `Characteristic_Integer`, `Characteristic_String` for compile-time type safety at the cost of API verbosity. Aligns with 1990s C best practices.

## Data Flow Through This File

1. **Load Phase** (map init): Server calls `BotLoadCharacter("bots/doom.cfg", 0.7)` → botlib parses file, interpolates traits for skill 0.7, returns opaque handle `h`.
2. **Runtime Phase** (per-frame AI): Game AI calls `Characteristic_BFloat(h, CHARACTERISTIC_AGGRESSIVENESS, 0.0, 1.0)` → botlib array-indexes into loaded profile, applies bounds, returns interpolated value.
3. **Unload Phase** (bot removal): Server calls `BotFreeCharacter(h)` → botlib deallocates internal `bot_character_t`.
4. **Shutdown Phase** (server close): `BotShutdownCharacters()` flushes all cached profiles at once.

## Learning Notes

**Era-Specific Design**: This interface epitomizes Q3A's data-driven 2000s FPS philosophy—characteristics are stored in text `.cfg` files, loaded at runtime, and blended for difficulty. Modern engines (Unreal, Unity) use ECS, scriptable properties, or behavior trees; Q3A baked FSMs and lookup tables.

**Skill Interpolation Pattern**: The skill parameter is a form of **difficulty-driven polymorphism**—one bot *personality* (file) supports many difficulty levels by interpolating characteristic values. This economical design reduced asset duplication (many fewer .cfg files needed).

**No Struct Definitions**: The caller never sees `struct bot_character_t` or characteristic index constants—they're internal to `code/botlib/`. The game module likely defines `#define CHARACTERISTIC_AGGRESSIVENESS 3` elsewhere and uses numeric indices. This strong coupling (game ↔ botlib on hard-coded indices) is a trade-off: simple, fast, but brittle if indices shift.

**Comparison to Modern Patterns**: 
- Modern engines use **ECS** (components on entities) or **property tables** (runtime intrection). Q3A uses a **handle-based object pool** hidden in botlib.
- Modern engines typically support **Hot Reload** of character data; Q3A requires `BotShutdownCharacters()` at server teardown (no mid-game reload).
- Modern engines expose **schema/metadata** so tools can validate .cfg files; Q3A trusts botlib's parser silently.

## Potential Issues

- **No documented characteristic index ranges**: Callers must know correct indices (e.g., `CHARACTERISTIC_AGGRESSIVENESS = ?`); a missing or wrong index call succeeds but returns garbage.
- **String buffer overflow guarded but not enforced**: `Characteristic_String(h, idx, buf, size)` copies into `buf` but no assert/error if `size` is too small; silent truncation possible.
- **Resource leak risk**: If caller forgets `BotFreeCharacter(h)` or exceeds max simultaneous bots, memory accumulates until server shutdown.
- **No validation of skill range**: `BotLoadCharacter(..., 2.5)` (out-of-bounds skill) accepted without error; interpolation may clamp silently or produce nonsensical blends depending on implementation.
- **No way to query loaded characteristics at runtime** (e.g., for debug/logging); read-only query-then-use pattern forces game code to redundantly call `Characteristic_*` for inspection.
