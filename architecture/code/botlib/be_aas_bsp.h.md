# code/botlib/be_aas_bsp.h

## File Purpose
Header for the AAS (Area Awareness System) BSP interface within the botlib. It declares functions for BSP-space collision, visibility, entity querying, and BSP model metadata used by the bot navigation system.

## Core Responsibilities
- Declare internal (AASINTERN) BSP file load/dump and entity-link management functions
- Expose public trace and point-contents queries into the BSP world
- Provide PVS/PHS visibility tests between world points
- Expose area connectivity queries for AAS routing
- Support entity enumeration and key-value (epair) property lookup on BSP entities

## Key Types / Data Structures
None defined here; types referenced but defined elsewhere.

| Name | Kind | Purpose |
|---|---|---|
| `bsp_trace_t` | struct (defined elsewhere) | Result of a BSP ray/box trace |
| `bsp_link_t` | struct (defined elsewhere) | Node in a linked list of BSP leaves an entity occupies |
| `vec3_t` | typedef (defined elsewhere) | 3D float vector |
| `qboolean` | typedef (defined elsewhere) | Boolean return type |

## Global / File-Static State
None.

## Key Functions / Methods

### AAS_LoadBSPFile *(AASINTERN)*
- **Signature:** `int AAS_LoadBSPFile(void)`
- **Purpose:** Loads the BSP file into AAS internal structures.
- **Inputs:** None (filename inferred from global state).
- **Outputs/Return:** Integer status (success/failure).
- **Side effects:** Allocates and populates global BSP data.
- **Calls:** Not inferable from this file.
- **Notes:** Only compiled when `AASINTERN` is defined.

### AAS_BSPLinkEntity *(AASINTERN)*
- **Signature:** `bsp_link_t *AAS_BSPLinkEntity(vec3_t absmins, vec3_t absmaxs, int entnum, int modelnum)`
- **Purpose:** Links an entity into BSP leaf nodes that its bounding box overlaps.
- **Inputs:** Absolute bounding box, entity number, model number.
- **Outputs/Return:** Pointer to head of the new `bsp_link_t` list.
- **Side effects:** Modifies BSP leaf occupancy lists.
- **Calls:** Not inferable from this file.

### AAS_Trace
- **Signature:** `bsp_trace_t AAS_Trace(vec3_t start, vec3_t mins, vec3_t maxs, vec3_t end, int passent, int contentmask)`
- **Purpose:** Traces a box through the BSP world, returning collision data.
- **Inputs:** Start/end points, box extents, entity to pass through, content filter mask.
- **Outputs/Return:** `bsp_trace_t` by value describing the first hit.
- **Side effects:** None (read-only query).
- **Calls:** Not inferable from this file.

### AAS_inPVS / AAS_inPHS
- **Signature:** `qboolean AAS_inPVS(vec3_t p1, vec3_t p2)` / `qboolean AAS_inPHS(vec3_t p1, vec3_t p2)`
- **Purpose:** Test whether `p2` falls within the Potentially Visible Set or Potentially Hearable Set of `p1`.
- **Inputs:** Two world-space points.
- **Outputs/Return:** `qtrue` if visible/hearable, `qfalse` otherwise.
- **Side effects:** None.

### AAS_BoxEntities
- **Signature:** `int AAS_BoxEntities(vec3_t absmins, vec3_t absmaxs, int *list, int maxcount)`
- **Purpose:** Collects entity numbers whose bounds intersect the given AABB.
- **Inputs:** Bounding box, output array pointer, max entries.
- **Outputs/Return:** Count of entities found; entity numbers written into `list`.
- **Side effects:** Writes to caller-supplied `list` array.

### AAS_ValueForBSPEpairKey / AAS_VectorForBSPEpairKey / AAS_FloatForBSPEpairKey / AAS_IntForBSPEpairKey
- **Signature:** Various; all take `int ent, char *key` plus typed output pointer.
- **Purpose:** Retrieve typed values from a BSP entity's key-value epair table.
- **Outputs/Return:** Integer (1 = found, 0 = not found); value written to output parameter.
- **Notes:** `MAX_EPAIRKEY 128` defines the maximum key string length.

## Control Flow Notes
This header is included by botlib source files. `AASINTERN` guards distinguish functions used only inside the botlib from those callable by bot AI game code. No frame-loop entry points are declared here; these are utility queries called on demand during bot navigation and entity tracking.

## External Dependencies
- `bsp_link_t`, `bsp_trace_t`, `vec3_t`, `qboolean` — defined elsewhere (likely `be_aas_def.h` / `q_shared.h`)
- `#define MAX_EPAIRKEY 128` — only constant defined in this file
- Implementation in `be_aas_bspq3.c` (defined elsewhere)
