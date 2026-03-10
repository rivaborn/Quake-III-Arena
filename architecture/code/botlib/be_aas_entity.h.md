# code/botlib/be_aas_entity.h

## File Purpose
Public and internal header for the AAS (Area Awareness System) entity subsystem within the Quake III botlib. It declares functions for querying and managing game entity state as it pertains to bot navigation and collision detection.

## Core Responsibilities
- Declares internal (AASINTERN-gated) entity lifecycle management functions (invalidate, unlink, reset, update)
- Exposes public API for querying entity spatial properties (origin, size, bounding box)
- Provides entity-to-AAS-area mapping for bot navigation queries
- Exposes entity type and model index accessors used by the bot AI layer

## Key Types / Data Structures
None defined in this file; types are defined elsewhere.

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_InvalidateEntities *(AASINTERN)*
- Signature: `void AAS_InvalidateEntities(void)`
- Purpose: Marks all tracked AAS entity records as invalid, likely at the start of a frame update cycle.
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies global AAS entity state array.
- Calls: Not inferable from this file.
- Notes: Only compiled when `AASINTERN` is defined.

### AAS_UnlinkInvalidEntities *(AASINTERN)*
- Signature: `void AAS_UnlinkInvalidEntities(void)`
- Purpose: Removes entities that were not updated this frame from AAS area/BSP link structures.
- Inputs: None
- Outputs/Return: None
- Side effects: Modifies AAS spatial link lists.
- Notes: Complements `AAS_InvalidateEntities` in a mark-and-sweep pattern.

### AAS_ResetEntityLinks *(AASINTERN)*
- Signature: `void AAS_ResetEntityLinks(void)`
- Purpose: Clears AAS area and BSP leaf pointers for all entities, forcing re-linking.
- Inputs: None
- Outputs/Return: None
- Side effects: Nulls spatial link pointers in entity records.

### AAS_UpdateEntity *(AASINTERN)*
- Signature: `int AAS_UpdateEntity(int ent, bot_entitystate_t *state)`
- Purpose: Updates an entity's AAS record with new state from the game, re-linking it spatially.
- Inputs: `ent` — entity index; `state` — pointer to new entity state.
- Outputs/Return: Integer status code (success/failure not inferable).
- Side effects: Writes to global AAS entity table; may relink entity into AAS areas/BSP leaves.

### AAS_EntityBSPData *(AASINTERN)*
- Signature: `void AAS_EntityBSPData(int entnum, bsp_entdata_t *entdata)`
- Purpose: Fills out BSP collision data for an entity used during trace/clip operations.
- Inputs: `entnum` — entity index; `entdata` — output structure.
- Outputs/Return: None (output via pointer).

### AAS_EntitySize
- Signature: `void AAS_EntitySize(int entnum, vec3_t mins, vec3_t maxs)`
- Purpose: Returns the axis-aligned bounding box of the entity.
- Inputs: `entnum` — entity index.
- Outputs/Return: `mins`, `maxs` filled with bounding box extents.

### AAS_BestReachableEntityArea
- Signature: `int AAS_BestReachableEntityArea(int entnum)`
- Purpose: Returns the AAS area index most suitable for pathfinding from/to this entity's position.
- Inputs: `entnum` — entity index.
- Outputs/Return: AAS area number.
- Notes: Critical for bot goal-setting; bridges entity world-space position to AAS topology.

### AAS_EntityInfo
- Signature: `void AAS_EntityInfo(int entnum, aas_entityinfo_t *info)`
- Purpose: Returns a snapshot of the full AAS entity info record.
- Inputs: `entnum` — entity index; `info` — output struct.
- Outputs/Return: None (output via pointer).

### AAS_NextEntity
- Signature: `int AAS_NextEntity(int entnum)`
- Purpose: Iterates over valid entities; returns the next entity index after `entnum`.
- Inputs: `entnum` — current entity index.
- Outputs/Return: Next valid entity index, or sentinel (0/-1) when exhausted.

---
**Notes on trivial accessors:** `AAS_EntityOrigin`, `AAS_EntityType`, `AAS_EntityModelindex`, and `AAS_EntityModelNum` / `AAS_OriginOfMoverWithModelNum` are simple property-getter stubs for origin vector, type enum, and model index/number lookups.

## Control Flow Notes
This header participates in the per-frame entity update pipeline: `AAS_InvalidateEntities` → game pushes updates via `AAS_UpdateEntity` → `AAS_UnlinkInvalidEntities` cleans stale entries. The public accessors are called on-demand by bot AI code during goal evaluation and reachability queries.

## External Dependencies
- `bot_entitystate_t` — defined in botlib/botlib.h or game interface headers
- `bsp_entdata_t` — defined in BSP/collision subsystem headers
- `aas_entityinfo_t` — defined in AAS internal headers (be_aas_def.h or aasfile.h)
- `vec3_t` — defined in q_shared.h
