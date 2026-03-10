# code/bspc/l_bsp_ent.h — Enhanced Analysis

## Architectural Role

This header defines the core entity and key-value data structures used during **offline BSP→AAS compilation** (BSPC toolchain). It serves as the bridge between the map file parser and AAS area generation, enabling BSPC to extract entity spatial data and configuration without linking to the game VM. Unlike runtime entity handling in `code/game/g_spawn.c` (which runs in the server VM), this file supports standalone offline preprocessing via tokenized script input, making it reusable across BSPC and botlib's compilation pipeline.

## Key Cross-References

### Incoming (who depends on this file)
- **BSPC compiler** (`code/bspc/be_aas_bspc.c`): Calls `ParseEntity` and `ParseEpair` to load `.map` or BSP entity lumps during AAS world compilation
- **Botlib AAS pipeline** (`code/botlib/be_aas_bspq3.c`): Has parallel entity parsing via `AAS_ParseBSPEntities`, using the same accessor pattern to query entity properties
- **Map utilities** (`code/bspc/aas_map.c`): Queries entities via `ValueForKey`, `FloatForKey` to extract brush ranges and areaportal configuration
- **Entity validation** (`code/bspc/aas_map.c`): Uses `wasdetail` and `areaportalnum` flags to classify geometry during AAS creation

### Outgoing (what this file depends on)
- **Script tokenizer** (`l_script.h`): Consumed by `ParseEntity`/`ParseEpair`; part of botlib's utility stack imported into BSPC
- **Math types** (`q_shared.h`): `vec3_t`, `vec_t`, `qboolean` foundational types
- **Utility functions** (`l_utils.h`, `l_memory.c`): Memory allocation for epair nodes; string utilities for `StripTrailing`

## Design Patterns & Rationale

**Linked-List Epair Storage**: `epair_t` uses singly-linked-list chaining rather than a hash table. This is appropriate for BSPC's offline context where entities are parsed once and queried infrequently during compilation, avoiding the overhead of hash-table construction.

**Typed Accessor Facade**: Rather than exposing raw epair list iteration, functions like `FloatForKey` and `GetVectorForKey` provide type-safe, null-safe lookups with sensible defaults (empty string, 0.0). This pattern appears identically in runtime game code, suggesting a deliberate portability strategy—the same entity interface can work both offline (BSPC) and at runtime (game VM).

**Dual-Target Design**: The file is designed for both:
1. **Offline BSPC compilation**: High-throughput entity parsing from map source
2. **Runtime botlib integration**: Entity queries during AAS pathfinding (via `botlib_import_t` entity callbacks)

This avoids code duplication and keeps the entity contract consistent across the pipeline.

## Data Flow Through This File

1. **Parse Phase** (offline, BSPC):
   - Script tokenizer provides `{ "key" "value" ... }` token stream
   - `ParseEntity` reads one entity block, calls `ParseEpair` for each key-value pair
   - Parsed entity written to global `entities[num_entities]`; `num_entities` incremented
   
2. **Query Phase** (compilation time):
   - AAS compiler iterates `entities[]` via `num_entities`
   - Calls `ValueForKey` (classname), `FloatForKey` (origin x/y/z), `GetVectorForKey` (target_position)
   - Extracts spatial/functional metadata for area portal registration and movement constraint modeling

3. **Runtime botlib Usage** (if AAS is loaded):
   - Entity queries happen through `botlib_import_t` callbacks, not direct struct access
   - Runtime never loads this header directly; access is mediated by botlib's BSP interface layer

## Learning Notes

**Offline Tool Architecture**: Quake III's toolchain separates preprocessing (BSPC, q3map) from runtime. This file exemplifies that split—entities are parsed once offline and baked into the AAS binary, not reparsed at runtime. This differs from modern engines (e.g., Unity, Unreal) where entity definitions are runtime-inspectable objects.

**String-Based Configuration**: The epair system relies entirely on string-based key-value lookups with implicit type coercion. There's no schema or validation—a missing key returns `""` or `0.0`, silently. This is era-appropriate for Quake (C-based, no reflection), but modern engines would use structured metadata or schema validation.

**Portability Pattern**: By keeping `entity_t` generic (no game-specific fields), the structure is reusable across BSPC, botlib, and the game VM. Game-specific extensions appear as optional epairs (e.g., `"team"`, `"model"`), avoiding header multiplication.

## Potential Issues

**Silent Type Conversion**: `FloatForKey` and `GetVectorForKey` coerce from string silently; malformed input (e.g., `"key" "not-a-number"`) returns `0.0` with no warning. The BSPC tool should validate entity sanity before use, but this header provides no protection.

**Max Entity Limit**: The `MAX_MAP_ENTITIES = 2048` hard limit is a BSP file format constraint, but there's no bounds checking in the header—`ParseEntity` assumes the caller prevents overflow of `entities[]`.
