# code/bspc/l_bsp_ent.c — Enhanced Analysis

## Architectural Role
This file implements entity parsing for the **offline BSPC compilation pipeline**, serving as the bridge between raw script tokens and the semantic entity objects consumed by AAS (Area Awareness System) generation. As a tool-side component (not runtime), it has a parallel in `code/botlib/be_aas_bspq3.c` (which parses runtime BSP entity strings), but operates at compile time with full BSP data available. The parsed `entities[]` global array feeds downstream AAS stages (`aas_map.c`, reachability analysis) that need entity properties like `classname`, `origin`, and model indices to classify terrain and compute bot-navigable areas.

## Key Cross-References
### Incoming (who depends on this file)
- **BSPC compilation pipeline** (e.g., `code/bspc/be_aas_bspc.c`, `aas_map.c`) invokes `ParseEntity` in a loop to populate the global `entities[]` array during BSP load
- **Downstream AAS stages** (area/face/reachability generation) query parsed entity properties via `ValueForKey`, `SetKeyValue`, `FloatForKey`, `GetVectorForKey` to determine terrain classification, obstacle placement, and trigger geometry
- **bspc.c (main entry point)** orchestrates BSP file loading, which triggers entity string parsing

### Outgoing (what this file depends on)
- **`botlib/l_script.h`** (`PS_ReadToken`, `PS_ExpectAnyToken`, `PS_UnreadLastToken`, `StripDoubleQuotes`) — lexical analysis and token stream management
- **`l_mem.h`** (`GetMemory`, `FreeMemory`, `copystring`) — heap allocation for entity and epair nodes
- **`l_cmd.h`** (`Error`, `qboolean`) — error handling and control flow
- **`l_math.h`** (`vec_t`, `vec3_t`) — vector types for spatial properties
- **C stdlib** (`strlen`, `strcmp`, `sscanf`, `atof`) — string and number parsing

## Design Patterns & Rationale
**Linked-list epairs with LIFO prepending**: Each `entity_t` holds a singly-linked `epair_t` chain. New pairs are prepended rather than appended, which is O(1) and matches typical parser accumulation patterns (reversing order if needed happens downstream or is irrelevant for key-value storage).

**Uniform string-based accessor interface**: `ValueForKey` returns all keys as strings; type-specific accessors (`FloatForKey`, `GetVectorForKey`) parse on demand. This avoids early type commitment and allows entities to evolve properties without schema changes—a pragmatic choice for offline tools.

**Global flat entity pool**: A pre-allocated `entities[MAX_MAP_ENTITIES]` array with a `num_entities` counter is simple and avoids dynamic resizing; appropriate for offline tools where map size is bounded.

**Eager bounds checking**: `MAX_KEY` (32) and `MAX_VALUE` (1024) limits are enforced at parse time, preventing stack overflows or unbounded allocations.

## Data Flow Through This File
1. **Input**: Script token stream (from `botlib/l_script.h` lexer), positioned at start of entity block
2. **Parsing**:
   - `ParseEntity` reads opening `{`, then loops calling `ParseEpair` until closing `}`
   - `ParseEpair` reads key and value tokens, strips quotes and trailing whitespace, allocates `epair_t`, returns it
   - Epairs are prepended to `entity_t.epairs` in parse order (LIFO)
3. **Output**: Global `entities[num_entities++]` populated; return `false` on EOF to signal end-of-stream
4. **Consumption**: Downstream AAS code queries `entities[i]` by entity index, calling `ValueForKey` to read classname, origin, angles, model numbers, etc.

## Learning Notes
**Era-appropriate C idioms**: No object-oriented encapsulation or generic hash tables; instead, linear O(N) linked-list lookup and manual string allocation. This reflects mid-2000s C practice and prioritizes simplicity over optimization in an offline tool.

**Separation of concerns**: Tokenization is delegated to `botlib/l_script.h`; this file focuses solely on entity/epair structure. Contrast with `code/botlib/be_aas_bspq3.c`, which parses pre-loaded BSP entity strings (already tokenized), showing how the same logical entity data can be sourced from different formats.

**Type conversion deferral**: `GetVectorForKey` uses `double` intermediates for `sscanf`, then assigns to `vec_t` (which may be `float` or `double`). This is defensive against size mismatches—a common pragmatism in cross-platform C.

**Tool vs. runtime duality**: This tool-side parser mirrors the runtime `AAS_ParseBSPEntities` pattern but operates on the full source BSP (with all entity classes and properties), whereas the runtime version processes a filtered game-server entity stream. Both exemplify the "entity as key-value store" pattern common in map formats (Quake, Half-Life, etc.).

## Potential Issues
**Linear lookup performance**: `ValueForKey` and `SetKeyValue` perform O(N) epair chain scans on every call. In large maps with many entities, this could add compile-time overhead. However, for an offline tool with typically <1000 entities and <100 properties each, this is acceptable.

**No epair uniqueness enforcement**: `SetKeyValue` prepends a new epair without checking for duplicates, risking orphaned old values if the linked list is not pruned. The implementation assumes proper cleanup elsewhere (or that duplicates are acceptable).
