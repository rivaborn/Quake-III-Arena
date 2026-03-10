# code/botlib/be_aas_bsp.h — Enhanced Analysis

## Architectural Role

This header is the **interface layer between the bot AI system and the raw BSP collision world**. It sits at a critical junction: game code and bot AI call these functions to query spatial and visibility information, while implementations delegate to the qcommon collision system (`CM_*` functions). The file effectively bridges two levels of abstraction—the high-level AAS area graph (which bots use for pathfinding) and the low-level BSP geometry (which qcommon manages).

## Key Cross-References

### Incoming (who depends on this file)
- **`be_aas_main.c`** — Calls `AAS_LoadBSPFile()` during botlib initialization; routes entity/trace queries through this interface
- **`be_aas_bspq3.c`** — Implements all functions declared here; actual BSP load logic and Q3-specific entity parsing
- **`be_aas_sample.c`** — Uses `AAS_Trace`, `AAS_PointContents`, `AAS_inPVS` for point-to-area mapping and reachability testing
- **`be_aas_entity.c`** — Calls `AAS_BSPLinkEntity`, `AAS_BoxEntities`, `AAS_UnlinkFromBSPLeaves` to track entity spatial occupancy
- **`be_aas_route.c`**, **`be_aas_reach.c`** — Use visibility queries (`AAS_inPVS`, `AAS_AreasConnected`) during reachability link computation
- **`code/game/g_*.c`** (server) — Via `trap_*` syscalls to engine, can indirectly use `AAS_Trace`, `AAS_PointContents`
- **`code/botlib/be_ai_*.c`** — AI decision modules (move, goal, weapon) query traces and visibility for pathfinding and target selection

### Outgoing (what this file depends on)
- **`qcommon/cm_*.c`** — Delegates all actual traces, PVS/area queries, and content testing to the collision system (via `botlib_import_t`)
- **`botlib_import_t`** interface — File loading, memory, BSP geometry queries, debug drawing (provided by the engine at runtime)
- **`be_aas_def.h`** — Type definitions (`bsp_trace_t`, `bsp_link_t`, `aas_entity_t`)

## Design Patterns & Rationale

1. **Guard with `AASINTERN`** — Functions like `AAS_LoadBSPFile`, `AAS_BSPLinkEntity`, `AAS_DumpBSPData` are only compiled when botlib builds internally (not exposed to game VMs). This enforces strict encapsulation: only the botlib module manages BSP entity linking.

2. **Lazy evaluation of spatial queries** — `AAS_Trace`, `AAS_PointContents`, `AAS_BoxEntities` are read-only query functions called on-demand during bot pathfinding, not pre-computed. This amortizes the cost across the bot decision loop.

3. **Dual-layer visibility caching** — `AAS_inPVS` and `AAS_inPHS` wrap qcommon's pre-computed PVS cluster data (built offline by the BSP compiler). Bots never compute visibility; they only query it.

4. **Entity spatial hashing via linked lists** — `AAS_BSPLinkEntity` inserts entities into `bsp_link_t` chains hanging off BSP leaf nodes. Fast box queries iterate leaves once; slow for sparse worlds, but Q3 maps are dense.

5. **Epair property access** — `AAS_ValueForBSPEpairKey`, `AAS_VectorForBSPEpairKey`, etc. support dynamic queries of entity spawn properties (e.g., `target`, `targetname` for teleporters). This avoids hard-coding Q3 entity schema into botlib.

## Data Flow Through This File

**Initialization (once per map load):**
- `AAS_LoadBSPFile()` → reads `.aas` file via `botlib_import` → populates `aasworld` singleton
- `AAS_ParseBSPEntities()` → extracts entity key-value pairs from BSP entity string lump

**Per-frame entity tracking:**
- Bot/player moves → `AAS_UpdateEntity()` → calls `AAS_BSPLinkEntity()` → entity inserted into BSP leaf `bsp_link_t` chains
- `AAS_BoxEntities()` iterates affected leaves, collects entity list

**Path planning queries:**
- Bot AI calls `AAS_Trace()` or `AAS_PointContents()` → delegates to `botlib_import.CM_Trace` / `CM_PointContents` → qcommon BSP system
- `AAS_inPVS(p1, p2)` → hashes points to PVS clusters → compares cluster bitmasks (precomputed by bspc)

## Learning Notes

- **Two-abstraction-level design** — Classic approach: raw geometry (BSP) exists in qcommon; AAS abstracts it for AI. Bots never directly use BSP functions; they call AAS, which calls BSP. This allows swapping the AI layer (e.g., to a different pathfinding algorithm) without touching collision code.
- **Visibility as a first-class query** — Q3 treats PVS not as a render optimization (as in older id engines) but as a general spatial visibility predicate available to AI and game code. This is idiomatic to Q3's architecture.
- **Entity property reflection** — The epair functions implement a dynamic property lookup without requiring a type system or scripting language. This is a lightweight alternative to VM-level entity attribute dictionaries (common in modern engines).
- **PVS cluster precomputation** — BSP compiler (bspc) computes PVS offline; q3map2 refines it. Runtime visibility queries are O(1) bit operations, not raycasts. A modern engine would likely compute this per-frame with GPU frustum culling.

## Potential Issues

- **Entity link fragmentation** — Entities that move frequently (projectiles, players) churn the `bsp_link_t` heap. No pooling or defragmentation strategy visible here; could cause cache misses in tight bot loops.
- **`MAX_EPAIRKEY` bound** — 128-character limit on epair keys may constrain mod authors adding long property names.
- **No input validation** — Functions like `AAS_Trace` accept raw entity/content masks with no range checks; invalid `passent` or `contentmask` values silently pass through to qcommon.
- **Visibility as static data** — PVS is precomputed; runtime geometry changes (moving doors, breakable walls) aren't reflected in visibility queries. Bots may not "see" through newly-opened portals.
