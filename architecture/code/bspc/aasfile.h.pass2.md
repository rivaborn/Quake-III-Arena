# code/bspc/aasfile.h — Enhanced Analysis

## Architectural Role

This file defines the **canonical binary serialization format** for Area Awareness System (AAS) navigation data, serving as the critical boundary contract between the offline BSP→AAS compiler (`bspc`) and the runtime bot navigation library (`botlib`). It spans two distinct semantic layers: immutable **3D geometry** (vertices, planes, edges, faces, convex areas, BSP tree) loaded from the BSP and built by `bspc/aas_create.c`, and **navigation metadata** (reachabilities, routing caches, cluster hierarchies) computed at load-time by `botlib/be_aas_reach.c` and `botlib/be_aas_route.c`. The 14-lump layout mirrors the BSP file format, enabling both tools to share common I/O patterns.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/bspc/` (offline compiler)**: `aas_create.c`, `aas_store.c`, `aas_file.c` populate and serialize all structures defined here into binary `.aas` files during map preprocessing.
- **`code/botlib/` (runtime library)**: `be_aas_file.c` deserializes lumps into the global `aasworld` singleton; `be_aas_reach.c`, `be_aas_route.c`, `be_aas_sample.c` navigate and query the in-memory structures every frame.
- **`code/game/` (server game VM)**: Indirectly depends via `code/botlib` syscalls (`trap_BotLib*` range); never includes this header directly—only botlib consumes the AAS format.
- **`code/botlib/be_aas_main.c`**: Core orchestrator that loads `.aas` files and initializes all subsystems that read the lumps.

### Outgoing (what this file depends on)

- **`q_shared.h`**: Sole external dependency, provides `vec3_t` typedef for all 3D vertex and vector fields.
- **No function calls**: This is a pure format definition; it declares no functions, only constants and data structure layouts.

## Design Patterns & Rationale

### 1. **Negative Indexing for Semantic Encoding**
- **`aas_edgeindex_t`** (typedef int): Negative index signals reversed vertex winding; avoids separate bool field, reducing structure size.
- **`aas_faceindex_t`** (typedef int): Negative index indicates "use backside of face normal"—critical for distinguishing convex area orientation without duplicating face records.
- **Rationale**: Early 2000s space/performance optimization. Saves memory and cache locality on disk and in RAM. Modern engines use explicit `bool reversed` fields for clarity.

### 2. **Discrete Presence Types (Not Bitflags)**
- `PRESENCE_NONE`, `PRESENCE_NORMAL`, `PRESENCE_CROUCH` are literal int constants (1, 2, 4), not `#define PRESENCE_NORMAL (1<<1)`.
- **Rationale**: Bots can only occupy **one** presence type per area at a time (standing or crouching, never both). Treating them as discrete eliminates confusion and catches logic errors (a bot queries "can I fit as PRESENCE_CROUCH here?", not a bitmask union).

### 3. **Fixed-Size Lumps with Explicit Offsets**
- `aas_lump_t` (fileofs + filelen) allows arbitrary lump ordering and in-place skip during partial loads.
- **Rationale**: Enables forward/backward compatibility if a future version adds lumps—old loaders skip unknown lumps, new loaders provide defaults for missing ones.

### 4. **Travel Type Extensibility**
- 32 travel types defined (`MAX_TRAVELTYPES`), but only ~19 actively used (WALK, JUMP, SWIM, ROCKET_JUMP, GRAPPLE, etc.).
- **Rationale**: Reserved space for mod authors and future variants (e.g., TRAVEL_WALLRUN, TRAVEL_DOUBLEWALL). Prevents ABI breakage if travel type enum grows.

### 5. **Cluster/Portal Hierarchy for Hierarchical Pathfinding**
- `aas_cluster_t` groups areas; `aas_portal_t` links clusters via transitional areas.
- **Rationale**: A* routing first navigates **clusters** (coarse), then **areas within a cluster** (fine). Reduces pathfinding state space by orders of magnitude on large maps (e.g., 10,000 areas → 50 clusters → 2–3 cluster steps).

## Data Flow Through This File

### Build-Time (offline)
```
Q3 .bsp file (BSP tree, faces, vertices)
    ↓ [bspc/aas_bspq3.c: AAS_LoadBSPFile]
Temporary in-memory AAS
    ↓ [bspc/aas_create.c: AAS_Create]
Convex area subdivision & face boundary extraction
    ↓ [bspc/aas_store.c: AAS_StoreFile]
14 lumps → binary .aas file on disk
```

### Runtime (every bot frame)
```
.aas file on disk
    ↓ [botlib/be_aas_file.c: AAS_LoadAASFile]
Deserialize into `aasworld_t` (global singleton)
    ↓ [botlib/be_aas_reach.c: AAS_InitReachability]
Compute all inter-area reachability links (14+ travel types)
    ↓ [botlib/be_aas_route.c: AAS_InitRouting]
Build cluster hierarchy & hierarchical routing caches (LRU evicted)
    ↓ [botlib/be_aas_main.c: AAS_Frame (per-tick)]
Each bot: AAS_PointAreaNum → area lookup → AAS_AreaTravelTimeToGoalArea (A* over clusters)
    ↓ [game/g_bot.c: bot_input_t synthesis]
Locomotion commands (FORWARD, JUMP, STRAFE) sent to game logic
```

---

## Learning Notes

### Idiomatic to Early 2000s Game Engines
- **Lump-based binary format**: Mirrors Quake 1/2/3 BSP design. Modern engines use JSON, YAML, or custom binary codecs with reflection.
- **Negative indexing**: Space-saving trick widespread in retro 3D engines. Modern engines prioritize clarity (explicit bool fields, enum types) over 4 bytes per reference.
- **Fixed-size structures**: No variable-length arrays in the binary format; all variable-length data (reachabilities, faces per area) is indirect via index ranges (`firstEdge + numEdges`). Modern engines often use offset-based serialization or schema-driven formats (FlatBuffers, Protocol Buffers).

### Presence Types as Discrete State
- The separation of `PRESENCE_NORMAL` and `PRESENCE_CROUCH` reflects the design: a **single convex area can support multiple presence types** (e.g., a tall room vs. a ventilation shaft), but a **single bot occupies exactly one** at any moment.
- Contrast with modern AI: navmesh systems (e.g., Recast) store walkability as bitmasks per polygon, allowing a single polygon to be walkable for multiple agent sizes.

### Reachability: The Navigation Bridge
- `aas_reachability_t` is the **fundamental unit of bot movement**: "from area X to area Y requires travel type Z, starting at point A, ending at point B, taking T milliseconds."
- No other engine construct defines inter-area movement. The entire bot pathfinding system is built on transitive closure of reachability links.
- 14+ travel types let bots reason about movement physics: a `TRAVEL_JUMP` reachability encodes the precise initial velocity and arc needed, while a `TRAVEL_ROCKET_JUMP` encodes weapon timing.

---

## Potential Issues

### 1. **Version Mismatch Hazard** (Low severity, mitigated by design)
- Header defines both `AASVERSION_OLD = 4` and `AASVERSION = 5`. If a map is compiled with version 5 but an old binary loads it with version 4 expectations, geometry fields will be misaligned.
- **Mitigation**: `be_aas_file.c:AAS_LoadAASFile` checks `header->version` and aborts if mismatch. No silent corruption.
- **Modern alternative**: Semantic versioning with per-field deprecation markers.

### 2. **32 Travel Types: Exhaustion Risk** (Theoretical)
- Mod authors adding new movement mechanics (wall-run, ledge-grab, wall-jump) will eventually exceed 32 types. Enum overflow wraps to zero (invalid), silently breaking pathfinding.
- **Mitigation**: Q3A mods rarely extended travel types; the design was conservative. Modern engines either use extensible enums (Rust `#[non_exhaustive]`) or string-keyed travel type names.

### 3. **No Checksum or Signature** (Data integrity)
- The file header has no CRC or SHA256 to detect corruption or version skew between `.aas` and `.bsp`. A stale `.aas` file will load without warning if BSP geometry changed.
- **Mitigation**: `aas_header_t` stores `bspchecksum` (MD4 of BSP geometry); `botlib/be_aas_file.c` compares it against the in-memory BSP. Mismatch triggers a reload.

---

## Summary

`aasfile.h` is the **serialization contract** that decouples the offline compilation pipeline (bspc) from runtime navigation (botlib). Its fixed, lump-based structure, discrete presence types, and 32-slot travel-type enum reflect careful design for both performance (negative indexing, cache locality) and extensibility (14 lumps, reserved travel type slots). The cluster/portal hierarchy and reachability link system are the engine's primary innovation for **hierarchical pathfinding at scale**—allowing 10,000 areas to be navigated in near-constant time through a coarse cluster graph. Modern AI engines use navmeshes and dynamic reimeshing; Quake III's static AAS precomputation trades flexibility for determinism and spatial efficiency.
