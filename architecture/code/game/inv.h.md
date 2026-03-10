# code/game/inv.h

## File Purpose
A pure C header defining integer constants for inventory slots, item model indices, and weapon indices used by the bot AI system. It serves as a shared lookup table mapping game items to numeric identifiers consumed by botlib's fuzzy logic and goal-evaluation scripts.

## Core Responsibilities
- Defines `INVENTORY_*` slot indices for armor, weapons, ammo, powerups, and flags/cubes used by bot AI inventory queries
- Defines enemy awareness constants (`ENEMY_HORIZONTAL_DIST`, `ENEMY_HEIGHT`, `NUM_VISIBLE_*`) as pseudo-inventory fuzzy inputs
- Defines `MODELINDEX_*` constants that must stay synchronized with the `bg_itemlist` array in `bg_misc.c`
- Defines `WEAPONINDEX_*` constants mapping logical weapon slots to 1-based integer IDs

## Key Types / Data Structures
None. This file contains only preprocessor `#define` constants.

## Global / File-Static State
None. Pure header with no variable declarations.

## Key Functions / Methods
None. This file contains no functions.

## Control Flow Notes
This file participates in no runtime control flow. It is a compile-time constant table included by:
- Bot AI source files (`ai_main.c`, `ai_dmq3.c`, `ai_goal.c`, etc.) to index into the bot's fuzzy logic inventory array
- Potentially botlib `.c` files that evaluate item pickup priority and weapon selection

The `INVENTORY_*` indices are used as array subscripts into the bot's internal item-presence/quantity state vector, queried each frame during bot decision-making. The `MODELINDEX_*` values correspond to positional indices in `bg_itemlist[]` and are used to identify spawned entities. `WEAPONINDEX_*` are 1-based and map to `WP_*` weapon enum values indirectly.

## External Dependencies
- **`bg_misc.c`** — `bg_itemlist[]` array ordering must exactly match the `MODELINDEX_*` sequence; a mismatch silently corrupts bot item recognition
- **`MISSIONPACK`** — conditional compilation guard present but body is empty (`#error` is commented out); mission pack items (`INVENTORY_KAMIKAZE`, `MODELINDEX_KAMIKAZE`, etc.) are defined unconditionally regardless of the guard

## Notes
- `INVENTORY_GAUNTLET` starts at index 4, leaving slots 2–3 unused/reserved
- `INVENTORY_BFG10K` skips index 12 (no `INVENTORY_` entry for 12), creating a gap
- All three index namespaces (`INVENTORY_*`, `MODELINDEX_*`, `WEAPONINDEX_*`) are independent and not interchangeable despite covering the same item set
