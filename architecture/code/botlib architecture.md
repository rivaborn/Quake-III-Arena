# Subsystem Overview

## Purpose
`botlib` is Quake III Arena's self-contained bot library, implementing the full navigation (AAS), pathfinding, movement, AI decision-making, and elementary action pipeline for bot clients. It is compiled as a separate module and exposes a versioned function-pointer table (`botlib_export_t`) to the engine, consuming engine services exclusively through a corresponding `botlib_import_t` callback table. All bot perception, route planning, weapon selection, chat, goal management, and frame-level input generation reside here.

## Key Files

| File | Role |
|---|---|
| `be_interface.c` / `be_interface.h` | Module entry point (`GetBotLibAPI`); owns `botimport` and `botlibglobals` singletons; wires all subsystem exports into `botlib_export_t` |
| `aasfile.h` | On-disk AAS binary format: magic, version, lump layout, all geometric and topological structs |
| `be_aas_def.h` | Central internal AAS header; defines monolithic `aas_t` world state, entity links, routing caches, settings; aggregates all AAS sub-headers |
| `be_aas_main.c` / `be_aas_main.h` | AAS lifecycle coordinator: setup, per-frame update, map load, shutdown, string/model index registry |
| `be_aas_file.c` / `be_aas_file.h` | AAS binary file load/write; endian byte-swapping; header XOR obfuscation |
| `be_aas_bspq3.c` / `be_aas_bsp.h` | BSP world interface; delegates traces/contents/PVS to `botimport`; owns BSP entity key-value store |
| `be_aas_sample.c` / `be_aas_sample.h` | AAS spatial queries: point-to-area BSP traversal, bounding-box sweep trace, area link heap management |
| `be_aas_reach.c` / `be_aas_reach.h` | Computes all inter-area reachability links (walk, jump, swim, ladder, teleport, elevator, weapon jump, etc.) |
| `be_aas_route.c` / `be_aas_route.h` | Dijkstra-like travel-time routing over cluster/portal graph; LRU routing cache; `.rcd` cache serialization |
| `be_aas_cluster.c` / `be_aas_cluster.h` | Partitions AAS areas into clusters via portal flood-fill; assigns cluster-local indices |
| `be_aas_move.c` / `be_aas_move.h` | Physics movement simulation: gravity, friction, stepping, swimming, ladder; trajectory prediction |
| `be_aas_entity.c` / `be_aas_entity.h` | Per-frame entity state sync into AAS spatial database; area/BSP-leaf linkage for dynamic objects |
| `be_aas_optimize.c` / `be_aas_optimize.h` | Post-load AAS geometry compaction: strips all non-ladder faces/edges/vertices; remaps reachability indices |
| `be_aas_routealt.c` / `be_aas_routealt.h` | Alternative route goal discovery: mid-range waypoint clustering for tactically varied paths |
| `be_aas_debug.c` / `be_aas_debug.h` | Debug visualization: renders AAS geometry, reachabilities, clusters via engine debug-line/polygon API |
| `be_ea.c` / `be_ea.h` | Elementary Actions layer: per-client `bot_input_t` buffers; translates bot decisions to move/view/attack/command input |
| `be_ai_move.c` / `be_ai_move.h` | Movement AI: per-bot state machine; selects next reachability link toward goal; executes all travel types |
| `be_ai_goal.c` / `be_ai_goal.h` | Goal AI: level item tracking, per-bot goal stacks, fuzzy-weight LTG/NBG selection, avoid-goal lists |
| `be_ai_weap.c` / `be_ai_weap.h` | Weapon AI: loads weapon/projectile configs; fuzzy-logic best-weapon selection per bot inventory |
| `be_ai_char.c` / `be_ai_char.h` | Bot character/personality system: loads skill-bracketed characteristic files; interpolates numeric traits |
| `be_ai_chat.c` / `be_ai_chat.h` | Chat AI: console message queues, synonym/template expansion, pattern matching, reply-chat key evaluation |
| `be_ai_gen.c` | Genetic selection: fitness-proportionate roulette-wheel parent/child selection for weight evolution |
| `be_ai_weight.c` / `be_ai_weight.h` | Fuzzy logic weight evaluation: parses `weight` config trees; deterministic and stochastic evaluation; genetic mutation/interbreeding |
| `l_memory.c` / `l_memory.h` | Memory abstraction: wraps `botimport.GetMemory`/`HunkAlloc` with magic-ID bookkeeping |
| `l_libvar.c` / `l_libvar.h` | Internal cvar-like system: named string/float variables independent of the engine cvar system |
| `l_script.c` / `l_script.h` | Lexer/tokenizer: loads script text from file or memory; produces typed tokens (string, number, name, punctuation) |
| `l_precomp.c` / `l_precomp.h` | C-like preprocessor: `#define`, `#include`, `#if`/`#ifdef` conditional compilation for bot config files |
| `l_struct.c` / `l_struct.h` | Generic struct serialization: schema-driven read/write of C structs from/to script sources and files |
| `l_log.c` / `l_log.h` | File-based logging: plain and timestamped writes gated by the `"log"` libvar |
| `l_crc.c` / `l_crc.h` | 16-bit CCITT CRC: stateful and one-shot checksumming for data integrity |
| `l_utils.h` | Utility macros: `Maximum`, `Minimum`, `MAX_PATH`, `vectoangles` alias |
| `be_aas_funcs.h` | Aggregation header: single include facade for all AAS sub-module headers (excluded in BSPC builds) |

## Core Responsibilities

- **Navigation data management:** Load, validate, byte-swap, and hold the AAS binary world (areas, edges, faces, planes, clusters, portals, reachabilities) in the global `aasworld` singleton; write it back and serialize routing caches on demand.
- **Spatial queries:** Provide point-to-area mapping, bounding-box sweep traces, multi-area line traces, entity-to-area linking, PVS/PHS tests, and point-contents queries against the AAS BSP tree.
- **Reachability computation:** Classify every inter-area movement transition (14+ travel types) into a compact reachability graph used by pathfinding.
- **Pathfinding and routing:** Compute and cache travel times across the cluster/portal hierarchy via Dijkstra-like relaxation; expose per-frame travel-time and reachability queries to the AI layer.
- **Movement physics simulation:** Simulate client movement (gravity, friction, acceleration, stepping, liquid) to evaluate jump arcs, weapon-jump trajectories, and reachability validity.
- **Bot AI pipeline:** Drive per-bot goal selection (item pickup, LTG/NBG fuzzy scoring), movement state machines (travel-type execution), weapon selection (fuzzy inventory scoring), chat (template matching and reply), and personality (characteristic interpolation).
- **Elementary action output:** Accumulate per-frame `bot_input_t` state (view angles, movement vector, action flags, console commands) and expose it for engine consumption.
- **Utility infrastructure:** Provide botlib-local memory, logging, libvar configuration, lexer, preprocessor, struct serialization, and CRC utilities that are independent of the engine's equivalent systems.

## Key Interfaces & Data Flow

**Exposed to the engine / game layer:**
- `GetBotLibAPI` — sole DLL entry point; returns a versioned `botlib_export_t` table containing three nested structs:
  - `aas` — AAS spatial query functions (`AreaTravelTime`, `Trace`, `PointAreaNum`, `EntityInfo`, etc.)
  - `ea` — Elementary Action functions (`EA_Move`, `EA_Jump`, `EA_Attack`, `EA_GetInput`, etc.)
  - `ai` — Higher-level AI functions (`BotChooseWeapon`, `BotChooseGoal`, `BotMoveToGoal`, `BotChatTest`, etc.)
- `botlib_export_t.BotLibSetup` / `BotLibShutdown` — lifecycle control called by the server
- `botlib_export_t.BotLibFrame` — per-frame tick advancing AAS deferred init and entity updates
- `botlib_export_t.BotLibLoadMap` — triggers BSP/AAS file loading and full subsystem reinitialization

**Consumed from the engine / game layer (via `botlib_import_t botimport`):**
- File system: `FS_FOpenFile`, `FS_Read`, `FS_Write`, `FS_FCloseFile`, `FS_Seek`
- Memory: `GetMemory`, `FreeMemory`, `HunkAlloc`, `AvailableMemory`
- BSP/collision: `Trace`, `PointContents`, `inPVS`, `inPHS`, `EntityData`, `BSPEntityData`
- Debug visualization: `DebugLineCreate`, `DebugLineShow`, `DebugLineDelete`, `DebugPolygonCreate`, `DebugPolygonDelete`
- Client commands: `BotClientCommand`
- Printing/error: `Print`
- Timing: `Milliseconds`
- Entity state: per-frame `BotEntityState` updates pushed by the server into `be_aas_entity.c`

**Internal data flow (subsystem to subsystem):**
```
Engine import (botimport)
  └─► be_aas_bspq3.c   (BSP traces, entity lump)
  └─► be_aas_file.c    (file I/O for .aas / .rcd)

aasworld (global aas_t)
  ◄── be_aas_file.c    (loaded geometry)
  ◄── be_aas_reach.c   (computed reachability graph)
  ◄── be_aas_cluster.c (cluster/portal assignments)
  ◄── be_aas_route.c   (routing caches)
  ◄── be_aas_optimize.c (compacted geometry arrays)
  ──► be_aas_sample.c  (spatial queries)
  ──► be_aas_move.c    (physics settings, area data)
  ──► be_aas_route.c   (travel time computation)

be_ai_goal.c   ──► AAS routing queries ──► be_aas_route.c
be_ai_move.c   ──► AAS routing + traces ──► be_aas_route.c / be_aas_sample.c
be_ai_move.c   ──► elementary actions  ──► be_ea.c
be_ai_weap.c   ──► fuzzy weights        ──► be_ai_weight.c
be_ai_chat.c   ──► AAS time             ──► be_aas_main.c
be_ai_char.c   ──► script/precomp       ──► l_script.c / l_precomp.c
```

## Runtime Role

**Initialization (`BotLibSetup` / `BotLibLoadMap`):**
1. `be_interface.c` validates API version and populates `botimport`.
2. `Export_BotLibSetup` initializes subsystems in order: AAS settings, EA (allocates input buffers), weapon AI, goal AI, chat AI, move AI.
3. On map load, `AAS_LoadBSPFile` pulls entity data via `botimport`; `AAS_LoadAASFile` reads and byte-swaps the `.aas` lump file.
4. `AAS_InitClustering`, `AAS_InitReachability`, `AAS_InitRouting`, `AAS_InitAlternativeRouting` are called; reachability computation is deferred across frames via `AAS_ContinueInitReachability`.

**Per-frame (`BotLibFrame` / per-bot AI calls):**
1. `AAS_Frame` advances the AAS clock, invalidates stale entities, relinks moved entities (`be_aas_entity.c`), and continues incremental reachability calculation if not yet complete.
2. Game code calls AI functions (goal selection, move-to-goal, weapon selection, chat) on each bot client slot.
3. `be_ai_move.c` queries routing via `be_aas_route.c`, selects the next reachability link, and calls `be_ea.c` action setters to build `bot_input_t`.
4. Game code calls `EA_GetInput` to retrieve accumulated input for the bot client; `EA_ResetInput` clears it for the next frame.

**Shutdown (`BotLibShutdown`):**
- Subsystems are torn down in reverse order; `AAS_Shutdown` frees all AAS arrays, routing caches, link heaps, and entity tables; `BotShutdownWeights`, `BotShutdownChat`, `BotShutdownGoal`, `BotShutdownMove` free their respective per-bot handle arrays; `EA_Shutdown` frees input buffers; `LibVarDeAllocAll` cleans up all libvars.

## Notable Implementation Details

- **`AASINTERN` preprocessor gate:** Internal AAS lifecycle and link-management functions are declared only when `AASINTERN` is defined, enforcing a public/private API boundary within a single compilation context rather than via separate translation units.
- **`BSPCINCLUDE` guard in `be_aas_funcs.h`:** The entire AAS function facade is excluded when building the offline BSPC map compiler tool, which shares AAS file-format code but not the runtime navigation stack.
- **Deferred reachability computation:** `AAS_ContinueInitReachability` spreads the O(N²) area-pair reachability scan across multiple server frames to avoid a single-frame stall on map load.
- **LRU routing cache with memory pressure eviction:** `be_aas_route.c` maintains ordered lists of area and portal routing cache entries and evicts oldest entries when `AvailableMemory` falls below a threshold, making the cache size adaptive to available memory.
- **Post-load geometry compaction (`be_aas_optimize.c`):** After reachability is computed, all AAS geometric data except ladder faces is discarded and arrays are reallocated at minimal size, reducing runtime memory footprint.
- **Dual memory pools:** All botlib allocations go through `l_memory.c`, which routes requests to either `botimport.GetMemory` (general heap, freed individually) or `botimport.HunkAlloc` (frame/map-scoped hunk, freed in bulk on map change). Magic-ID words guard against double-free and mismatched pool frees.
- **AAS file header XOR obfuscation (`AAS_DData`):** The AAS file header is lightly obfuscated with a XOR cipher; this is not cryptographic protection but a simple integrity/version fingerprint mechanism inherited from the original tool pipeline.
- **`botlib_export_t` versioning:** `GetBotLibAPI` rejects calls with a mismatched `BOTLIB_API_VERSION`, ensuring the engine and bot library remain ABI-compatible across rebuilds.
- **Fuzzy weight genetic evolution:** `be_ai_weight.c` and `be_ai_gen.c` implement a fitness-proportionate roulette-wheel selection and crossover/mutation system, allowing bot AI parameters to evolve at runtime — an unusual feature for a shipped game engine.
- **Language:** All botlib source is ANSI C (`.c`/`.h`); it targets both native compilation (linked into the engine or as a DLL) and shares data-structure headers with the BSPC offline map compiler tool.
