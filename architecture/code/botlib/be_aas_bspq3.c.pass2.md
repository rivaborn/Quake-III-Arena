# code/botlib/be_aas_bspq3.c — Enhanced Analysis

## Architectural Role

This file is the **runtime BSP/entity gateway** for botlib, bridging AAS (Area Awareness System) navigation and entity-aware pathfinding to the Q3 engine's collision and entity subsystems. It solves a critical architectural problem: botlib is delivered as a self-contained library with no direct engine linkage, yet needs real-time BSP traces, PVS visibility checks, and entity metadata. The solution is a callback-based abstraction layer (`botlib_import_t`) that the engine populates at startup; this file owns both the wrapper implementations and the cached BSP entity data store that serves upper AAS layers.

## Key Cross-References

### Incoming (who depends on this file)
- **be_aas_main.c**: Calls `AAS_LoadBSPFile()` during map initialization and `AAS_DumpBSPData()` on unload (lifecycle management).
- **be_aas_entity.c**: Calls entity accessor functions (`AAS_ValueForBSPEpairKey`, `AAS_VectorForBSPEpairKey`, etc.) to extract spawn properties (origin, angles, target, health) from map entities for movement/reachability computation.
- **be_aas_move.c / be_aas_sample.c**: Call `AAS_Trace()` and `AAS_PointContents()` heavily during movement simulation and area sampling.
- **be_aas_route.c**: Calls `AAS_inPVS()` for optional visibility-aware pathfinding.
- **be_aas_reach.c**: Uses entity queries to validate jump reachability to jump pads, teleporters, elevators, and ladder/water volumes.

### Outgoing (what this file depends on)
- **botimport vtable** (defined in be_interface.c, populated by server at startup):
  - `botimport.Trace()` — delegated by `AAS_Trace()`; implements swept AABB collision
  - `botimport.PointContents()` — point-in-BSP-solid tests
  - `botimport.EntityTrace()` — per-entity AABB traces
  - `botimport.inPVS()` — cluster visibility tests
  - `botimport.BSPEntityData()` — raw entity lump text fetch
  - `botimport.BSPModelMinsMaxsOrigin()` — bmodel AABB queries
  - `botimport.Print()` — debug output
- **Memory layer** (l_memory.h): Hunk allocation for persistent entity data.
- **Script parser** (l_script.h): Tokenization of entity text.

## Design Patterns & Rationale

**Thin Abstraction via Callbacks**: Most functions are 1–2 line pass-throughs to `botimport` callbacks. Why?
- **Decoupling**: botlib.dll/.so never links against the engine; the import table is the contract.
- **Platform agnostic**: Different game ports (Q3A, TA, WoLF, ET) can swap implementations of BSP/entity behavior without recompiling botlib.
- **Testability**: Tests can mock the `botlib_import_t` without a full engine.

**Entity Parsing & Caching**: Why load and parse entity data into `bspworld`?
- **Determinism**: Parsed data can be logged/replayed for debugging without engine state.
- **Performance**: Entity properties are accessed once-per-load, not per-frame; caching trades memory for speed.
- **Robustness**: Local parsing allows graceful error recovery and custom validation.

**Stub Functions** (`AAS_UnlinkFromBSPLeaves`, `AAS_BSPLinkEntity`, `AAS_BoxEntities`):
- These exist for interface contract compliance; they're inherited from the BSPC offline compiler, which *does* implement spatial entity indexing.
- Q3 runtime doesn't use them: the engine's sector tree handles entity spatial queries; botlib only traces and samples.
- Keeping them preserves the `be_aas_bsp.h` interface across tools and runtime.

## Data Flow Through This File

**Level Load**:
```
Engine calls AAS_LoadBSPFile()
  → AAS_DumpBSPData() [cleanup prior data]
  → botimport.BSPEntityData() [fetch raw "{...} {...}" text]
  → GetClearedHunkMemory() [allocate buffer]
  → Com_Memcpy() [copy text]
  → AAS_ParseBSPEntities()
    → LoadScriptMemory() [tokenize]
    → PS_ReadToken() loop [parse entities]
      → GetHunkMemory() [allocate epair nodes & strings]
      → Form linked-list tree: bspworld.entities[i].epairs
  → bspworld.loaded = qtrue
```

**Runtime Entity Queries**:
```
be_aas_entity.c / be_aas_reach.c call:
  AAS_VectorForBSPEpairKey(ent, "origin", ...)
    → AAS_ValueForBSPEpairKey(ent, "origin", buf, ...)
      → Linear search: bspworld.entities[ent].epairs->next->...
      → strcmp(epair->key, "origin")
      → Return epair->value, sscanf() in caller
```

**Movement/Spatial**:
```
be_aas_move.c / be_aas_sample.c call:
  AAS_Trace(start, mins, maxs, end, ...)
    → botimport.Trace(...) [engine performs actual swept collision]
    → return bsp_trace_t with hitpoint, fraction, surfaceFlags, ...
```

**Level Unload**:
```
Engine calls AAS_DumpBSPData()
  → AAS_FreeBSPEntities() [loop epairs, FreeMemory each key/value/node]
  → FreeMemory(bspworld.dentdata) [text buffer]
  → Com_Memset(&bspworld, 0, ...)
```

## Learning Notes

**For a developer studying this engine:**
- **Callback-based architecture**: This is a clean inversion-of-control pattern—botlib knows *nothing* about the engine; the engine provides all services. Modern analogs: dependency injection, MicroKernels, plugin APIs.
- **Quake-era entity data**: The `key = "value"` text format is remarkably flexible and human-editable. Modern engines use JSON, YAML, or binary serialization—but this approach's simplicity endured through Q3, QL, Wolfenstein, ET, and Doom 3.
- **Hunk allocation philosophy**: All entity data is allocated at load-time from a single linear hunk; freed en masse at unload. No per-frame allocations. This matches the era's performance constraints and simplifies memory profiling.
- **Script parsing as infrastructure**: The same tokenizer handles entity data, config files, and AI scripts. Reusable infrastructure pattern.

**Idiomatic to this era:**
- No virtual methods, inheritance, or runtime type information.
- POD (Plain Old Data) structures; no invariant enforcement.
- Bit-packed flags and macros instead of enums.
- Callback tables instead of vtables.

**Connection to broader engine concepts:**
- **Data Source Layer** (layered architecture): abstracts away BSP lumps and engine callbacks.
- **Caching Pattern**: entity data is read-once, used many-times during pathfinding.
- **Dependency Injection**: `botlib_import_t` injected at initialization, enabling testability and modularity.
- **Anti-pattern avoidance**: Botlib does *not* try to maintain a mirror of the server's entity state; it only queries static spawn properties, avoiding sync bugs.

## Potential Issues

1. **PHS always returns true** (`AAS_inPHS`): Bots assume all entities are "audible" even if PVS-occluded. This is a correctness simplification (PHS computation is expensive) but means bots can theoretically navigate toward distant entities they shouldn't "hear." In practice, goal selection logic filters unreachable areas, masking the issue.

2. **Entity lookup failures are silent**: If `AAS_ValueForBSPEpairKey()` is called on an invalid entity or missing key, it returns `qfalse`. Callers *must* check the return value; if they don't (e.g., caller assumes key exists and reads garbage from uninitialized output), pathfinding may silently use wrong data. The function prints a warning for out-of-range entities, but not for missing keys.

3. **No entity change detection**: Once loaded, entity data is never re-read. If the server repositions a teleporter or changes a trigger, botlib still sees the old data until the next map load. This is by design (entities are assumed static), but level-streaming or dynamic entity loads would break silently.
