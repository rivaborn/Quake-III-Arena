# code/botlib/be_aas_entity.c

## File Purpose
Manages the AAS (Area Awareness System) entity table for the Quake III botlib. It synchronizes game-world entity state into the AAS spatial database, maintaining entity-to-AAS-area and entity-to-BSP-leaf linkages so the bot pathfinding system can reason about dynamic objects.

## Core Responsibilities
- Accept per-frame entity state updates and mirror them into `aasworld.entities[]`
- Detect changes in origin, angles, or bounding box to trigger spatial relinking
- Link/unlink entities to AAS areas and BSP leaf nodes
- Provide read accessors for entity origin, size, type, model index, and BSP data
- Iterate over valid entities (`AAS_NextEntity`) and find nearest entity by model
- Reset or invalidate the entity table between map loads or frames

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `aas_entity_t` | struct (defined in `be_aas_def.h`) | Per-entity AAS record: info block (`i`), area link list (`areas`), BSP leaf link list (`leaves`) |
| `aas_entityinfo_t` | struct (defined in `be_aas_def.h`) | Public entity info snapshot: origin, angles, mins/maxs, type, flags, solid, modelindex, animation state |
| `bot_entitystate_t` | struct (defined in `be_aas.h`) | Incoming state from the game: new origin, angles, bounds, solid type, model, animation fields |
| `bsp_entdata_t` | struct | Packed BSP-facing data: origin, angles, absolute mins/maxs, solid, modelnum |
| `ET_*` (enum) | enum (file-local) | Entity type constants: GENERAL, PLAYER, ITEM, MISSILE, MOVER |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `aasworld` | `aas_world_t` (extern) | global | The singleton AAS world state; owns `entities[]`, `maxentities`, `loaded`, `initialized`, `numframes` |
| `botimport` | `botlib_import_t` (extern) | global | Engine import table; used here for `Print` and via called functions |

## Key Functions / Methods

### AAS_UpdateEntity
- **Signature:** `int AAS_UpdateEntity(int entnum, bot_entitystate_t *state)`
- **Purpose:** Primary per-frame update. Copies incoming state into `aasworld.entities[entnum]`, detects spatial changes, and re-links the entity into AAS areas and BSP leaves when needed.
- **Inputs:** `entnum` — entity index; `state` — new state, or NULL to unlink/remove
- **Outputs/Return:** `BLERR_NOERROR` on success, `BLERR_NOAASFILE` if AAS not loaded
- **Side effects:** Mutates `aasworld.entities[entnum].i`; calls `AAS_UnlinkFromAreas`, `AAS_UnlinkFromBSPLeaves`, `AAS_LinkEntityClientBBox`, `AAS_BSPLinkEntity`, `AAS_BSPModelMinsMaxsOrigin`
- **Calls:** `AAS_Time`, `AAS_UnlinkFromAreas`, `AAS_UnlinkFromBSPLeaves`, `AAS_LinkEntityClientBBox`, `AAS_BSPLinkEntity`, `AAS_BSPModelMinsMaxsOrigin`, `botimport.Print`
- **Notes:** `relink` is forced true on frame 1. BSP-solid movers have their bounds recomputed from the model; bbox-solid entities use the state-supplied bounds. `ENTITYNUM_WORLD` is never spatially linked.

### AAS_EntityInfo
- **Signature:** `void AAS_EntityInfo(int entnum, aas_entityinfo_t *info)`
- **Purpose:** Copies the internal entity info block to the caller's buffer; primary read path for bot AI.
- **Inputs:** `entnum`, output pointer `info`
- **Outputs/Return:** Fills `*info`; zeroes it on error
- **Side effects:** None
- **Notes:** Guards against uninitialized world and out-of-range entnum.

### AAS_OriginOfMoverWithModelNum
- **Signature:** `int AAS_OriginOfMoverWithModelNum(int modelnum, vec3_t origin)`
- **Purpose:** Linear scan to find an `ET_MOVER` entity using a given model index and return its origin.
- **Inputs:** `modelnum` — BSP inline model number; `origin` — output
- **Outputs/Return:** `qtrue` if found, `qfalse` otherwise
- **Side effects:** None

### AAS_NearestEntity
- **Signature:** `int AAS_NearestEntity(vec3_t origin, int modelindex)`
- **Purpose:** Finds the entity with `modelindex` closest to `origin`, with an early-out XY proximity filter (±40 units).
- **Inputs:** Reference `origin`, target `modelindex`
- **Outputs/Return:** Entity number of best match, or 0
- **Notes:** The ±40 unit XY prefilter may miss legitimate candidates when the search space is sparse; comment-free optimisation heuristic.

### AAS_ResetEntityLinks / AAS_InvalidateEntities / AAS_UnlinkInvalidEntities
- Bulk maintenance helpers: reset area/leaf pointers to NULL, mark all entities invalid, and unlink entities whose `valid` flag is false, respectively.

### AAS_BestReachableEntityArea
- **Signature:** `int AAS_BestReachableEntityArea(int entnum)`
- **Purpose:** Delegates to `AAS_BestReachableLinkArea` with the entity's area link list; used by AI goal selection.

### AAS_NextEntity
- **Signature:** `int AAS_NextEntity(int entnum)`
- **Purpose:** Iterates to the next valid entity after `entnum`; returns 0 when exhausted.

## Control Flow Notes
Called during the bot server frame update sequence. `AAS_UpdateEntity` is called once per game entity per frame (from `be_aas_main.c` / engine glue). `AAS_InvalidateEntities` is called before the update pass; `AAS_UnlinkInvalidEntities` is called after, so that entities not refreshed this frame are removed from spatial structures.

## External Dependencies
- `../game/q_shared.h` — math macros (`VectorCopy`, `VectorAdd`, etc.), `qboolean`, `vec3_t`
- `be_aas_def.h` — `aasworld` global, `aas_entity_t`, `aas_world_t`
- `be_aas_funcs.h` — `AAS_UnlinkFromAreas`, `AAS_UnlinkFromBSPLeaves`, `AAS_LinkEntityClientBBox`, `AAS_BSPLinkEntity`, `AAS_BSPModelMinsMaxsOrigin`, `AAS_BestReachableLinkArea`, `AAS_Time`
- `be_interface.h` — `botimport` global
- `../game/botlib.h` — `BLERR_*` error codes, `bot_entitystate_t`, `PRESENCE_NORMAL`
- `../game/be_aas.h` — `bsp_entdata_t`, `aas_entityinfo_t` public types
- `aasfile.h` — AAS file format constants (included transitively)
