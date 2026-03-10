# code/bspc/writebsp.c

## File Purpose
Serializes the in-memory BSP tree and associated map data (planes, faces, leaves, nodes, brushes, models) into the flat arrays used by the BSP file format. It is the final output stage of the BSPC compiler before the binary file is written to disk.

## Core Responsibilities
- Walk the BSP node tree recursively and emit nodes, leaves, and faces into output arrays
- Emit planes, brushes, and brush sides (including axis-aligned bevel planes for collision)
- Assign leaf-face and leaf-brush index ranges
- Bookend model compilation with `BeginModel`/`EndModel` to track face/leaf ranges per entity model
- Assign `model` key-value pairs to brush entities (`SetModelNumbers`)
- Assign unique light style numbers to targetname-controlled lights (`SetLightStyles`)
- Initialize and finalize global BSP file counters (`BeginBSPFile`/`EndBSPFile`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `node_t` | struct (defined in qbsp.h) | BSP tree node (or leaf when `planenum == PLANENUM_LEAF`) |
| `face_t` | struct (defined in qbsp.h) | Polygon face with merge/split chain tracking |
| `bspbrush_t` | struct (defined in qbsp.h) | BSP-space brush fragment with sides |
| `mapbrush_t` | struct (defined in qbsp.h) | Original map brush with original sides |
| `portal_t` | struct (defined in qbsp.h) | Portal between two BSP nodes, carries output faces |
| `dnode_t`, `dleaf_t`, `dface_t`, `dplane_t`, `dbrush_t`, `dbrushside_t`, `dmodel_t` | structs (q2files.h) | Flat BSP file output structures |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `c_nofaces` | `int` | global | Count of nodes emitted with no faces |
| `c_facenodes` | `int` | global | Count of nodes emitted with faces |
| `planeused` | `int[MAX_MAP_PLANES]` | file-static (global array) | Reference count per plane, incremented when a node uses that plane |
| `firstmodleaf` | `int` | global | Leaf index at the start of the current model |

## Key Functions / Methods

### EmitPlanes
- **Signature:** `void EmitPlanes(void)`
- **Purpose:** Copies all map planes from `mapplanes[]` into `dplanes[]` output array.
- **Inputs:** Global `mapplanes`, `nummapplanes`
- **Outputs/Return:** Populates `dplanes[]`, increments `numplanes`
- **Side effects:** Writes to `dplanes`, increments `numplanes`; calls `Error` on overflow
- **Calls:** `VectorCopy`, `Error`
- **Notes:** No plane culling occurs; all original map planes are emitted unconditionally.

### EmitFace
- **Signature:** `void EmitFace(face_t *f)`
- **Purpose:** Emits a single final face into `dfaces[]` and its edge indices into `dsurfedges[]`.
- **Inputs:** `face_t *f` — must be a final (non-merged, non-split) face with ≥3 points
- **Outputs/Return:** Sets `f->outputnumber`; populates `dfaces[]` and `dsurfedges[]`
- **Side effects:** Increments `numfaces`, `numsurfedges`
- **Calls:** `GetEdge2`, `Error`
- **Notes:** Degenerate faces (< 3 points) or intermediate merged/split faces are silently skipped with `outputnumber = -1`.

### EmitMarkFace
- **Signature:** `void EmitMarkFace(dleaf_t *leaf_p, face_t *f)`
- **Purpose:** Recursively follows merge/split chains to add the final face's output number to `dleaffaces[]` for the given leaf, deduplicating entries.
- **Inputs:** `leaf_p` — leaf being populated; `f` — face (may be merged or split)
- **Outputs/Return:** Appends to `dleaffaces[]`
- **Side effects:** Increments `numleaffaces`; calls `Error` on overflow
- **Calls:** Recursive `EmitMarkFace`, `Error`

### EmitLeaf
- **Signature:** `void EmitLeaf(node_t *node)`
- **Purpose:** Emits a BSP leaf into `dleafs[]`, including its brush list and face list (skips faces for solid leaves).
- **Inputs:** `node_t *node` — must be a leaf node
- **Outputs/Return:** Populates `dleafs[]`, `dleafbrushes[]`, `dleaffaces[]`
- **Side effects:** Increments `numleafs`, `numleafbrushes`, `numleaffaces`
- **Calls:** `VectorCopy`, `EmitMarkFace`, `Error`
- **Notes:** Solid leaves (`CONTENTS_SOLID`) skip leafface emission.

### EmitDrawNode_r
- **Signature:** `int EmitDrawNode_r(node_t *node)`
- **Purpose:** Recursively walks the BSP tree, emitting internal nodes via `dnodes[]` and leaves via `EmitLeaf`. Tracks plane usage in `planeused[]`.
- **Inputs:** `node_t *node` — root of subtree
- **Outputs/Return:** Returns index into `dnodes[]` for nodes; returns `-(numleafs)` for leaves
- **Side effects:** Increments `numnodes`; calls `EmitFace` for all faces on a node; calls `EmitLeaf` for children that are leaves
- **Calls:** `EmitLeaf`, `EmitFace`, `VectorCopy`, `Error`, recursive `EmitDrawNode_r`
- **Notes:** Validates that plane numbers are even (never odd); child indices follow Q2 BSP convention: positive = node index, negative = `-(leaf+1)`.

### WriteBSP
- **Signature:** `void WriteBSP(node_t *headnode)`
- **Purpose:** Top-level entry point for BSP tree serialization; drives `EmitDrawNode_r` and area portal emission.
- **Inputs:** `node_t *headnode`
- **Side effects:** Resets `c_nofaces`/`c_facenodes`; calls `EmitDrawNode_r`, `EmitAreaPortals`; prints statistics
- **Calls:** `EmitDrawNode_r`, `EmitAreaPortals`, `qprintf`

### EmitBrushes
- **Signature:** `void EmitBrushes(void)`
- **Purpose:** Copies all map brushes and their sides into `dbrushes[]`/`dbrushsides[]`, then appends axis-aligned bevel planes not already present to support correct collision detection.
- **Inputs:** Global `mapbrushes`, `nummapbrushes`
- **Side effects:** Writes `dbrushes[]`, `dbrushsides[]`; increments `numbrushsides`, `numbrushes`
- **Calls:** `FindFloatPlane`, `VectorCopy`, `Error`

### BeginBSPFile / EndBSPFile
- **Signature:** `void BeginBSPFile(void)` / `void EndBSPFile(void)`
- **Purpose:** Initialize BSP output counters to known-valid state (reserving slot 0 for error sentinels); finalize by calling `EmitBrushes`, `EmitPlanes`, and entity string serialization.
- **Side effects:** Resets global counters; `EndBSPFile` calls `Q2_UnparseEntities`

### BeginModel / EndModel
- **Signature:** `void BeginModel(void)` / `void EndModel(void)`
- **Purpose:** Bracket the emission of a single entity submodel: record `firstface`, compute AABB from brush extents, then finalize `numfaces` count.
- **Calls:** `ClearBounds`, `AddPointToBounds`, `VectorCopy`, `Error`

### SetModelNumbers
- **Signature:** `void SetModelNumbers(void)`
- **Purpose:** Assigns `"model" "*N"` key-value to each brush entity with brushes.
- **Calls:** `SetKeyValue`, `sprintf`

### SetLightStyles
- **Signature:** `void SetLightStyles(void)`
- **Purpose:** Assigns unique light style numbers (32+) to all `light` entities that have a `targetname`, so animated lights work correctly.
- **Calls:** `ValueForKey`, `Q_strncasecmp`, `strcmp`, `strcpy`, `sprintf`, `SetKeyValue`, `Error`
- **Notes:** Limit of `MAX_SWITCHED_LIGHTS` (32) distinct targetnames; style indices start at 32.

## Control Flow Notes
This file is active during the **BSP compilation output phase**. The typical call sequence is:

1. `BeginBSPFile` — reset counters
2. Per-entity loop: `BeginModel` → `WriteBSP` → `EndModel`
3. `SetModelNumbers`, `SetLightStyles` — entity key assignment
4. `EndBSPFile` — flush brushes, planes, entity strings

It does not participate in rendering or runtime; it is exclusively a compiler-time serialization module within the BSPC tool.

## External Dependencies
- **Includes:** `qbsp.h` (pulls in all BSPC-internal headers, q2files.h for BSP output types, l_bsp_ent.h for entity helpers)
- **Defined elsewhere:**
  - `dplanes[]`, `dfaces[]`, `dleafs[]`, `dnodes[]`, `dbrushes[]`, `dbrushsides[]`, `dleaffaces[]`, `dleafbrushes[]`, `dsurfedges[]`, `dmodels[]`, `numplanes`, `numfaces`, `numleafs`, `numnodes`, `numbrushes`, `numbrushsides`, `numleaffaces`, `numleafbrushes`, `numsurfedges`, `numedges`, `numvertexes`, `nummodels` — global BSP output arrays/counters (q2files.h / bspc global state)
  - `mapplanes`, `nummapplanes`, `mapbrushes`, `nummapbrushes`, `entities`, `num_entities`, `entity_num` — map loading globals
  - `GetEdge2` — edge deduplication (faces.c or similar)
  - `EmitAreaPortals` — portals.c
  - `Q2_UnparseEntities` — map_q2.c
  - `FindFloatPlane` — map.c
  - `ValueForKey`, `SetKeyValue` — l_bsp_ent.c
  - `qprintf`, `Error` — l_cmd.c
