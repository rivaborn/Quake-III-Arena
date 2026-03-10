# code/bspc/aas_create.c

## File Purpose
Converts a BSP tree (produced by the BSPC map compiler) into a temporary AAS (Area Awareness System) world representation. It manages the full AAS creation pipeline from BSP leaf extraction through face classification, area merging, subdivision, and final file storage.

## Core Responsibilities
- Allocate and free temporary AAS data structures (faces, areas, nodes, node buffers)
- Convert BSP leaf nodes and their portals into convex AAS areas with classified faces
- Classify faces as ground, gap, solid, liquid, liquid-surface, or ladder
- Validate and repair face winding orientation relative to area centers
- Orchestrate the multi-pass AAS build pipeline in `AAS_Create`
- Assign area settings (flags, presence type, contents) from aggregated face flags
- Remove degenerate geometry (tiny faces, collinear winding points)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `tmp_face_t` | struct (typedef) | Temporary AAS face with winding, plane, face flags, and doubly-linked per-area lists |
| `tmp_area_t` | struct (typedef) | Temporary convex AAS area with linked face list, contents, presence type, and settings |
| `tmp_node_t` | struct (typedef) | Temporary BSP node referencing an area or branching to children |
| `tmp_nodebuf_t` | struct (typedef) | Pooled buffer of 128 `tmp_node_t` to amortize allocations |
| `tmp_aas_t` | struct (typedef) | Root container for all temporary faces, areas, nodes, and the node buffer chain |
| `tmp_areasettings_t` | struct (typedef) | Area settings: contents, flags, presence type, model number |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `tmpaasworld` | `tmp_aas_t` | global | Singleton holding all in-progress temporary AAS data during conversion |

## Key Functions / Methods

### AAS_InitTmpAAS
- **Signature:** `void AAS_InitTmpAAS(void)`
- **Purpose:** Zero-initializes `tmpaasworld` before conversion begins.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Resets all fields of `tmpaasworld`.
- **Calls:** None
- **Notes:** Must be called before any alloc functions.

### AAS_FreeTmpAAS
- **Signature:** `void AAS_FreeTmpAAS(void)`
- **Purpose:** Frees all temporary faces, areas, and node buffers.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Calls `FreeWinding`, `FreeMemory` on all linked-list members.
- **Calls:** `FreeWinding`, `FreeMemory`

### AAS_AllocTmpFace / AAS_FreeTmpFace
- **Signature:** `tmp_face_t *AAS_AllocTmpFace(void)` / `void AAS_FreeTmpFace(tmp_face_t *)`
- **Purpose:** Allocate/free a face and maintain the global doubly-linked face list.
- **Side effects:** Modifies `tmpaasworld.faces`, `numfaces`, `facenum`.

### AAS_AllocTmpArea / AAS_FreeTmpArea
- **Signature:** `tmp_area_t *AAS_AllocTmpArea(void)` / `void AAS_FreeTmpArea(tmp_area_t *)`
- **Purpose:** Allocate/free an area within the global doubly-linked area list.
- **Side effects:** Modifies `tmpaasworld.areas`, `numareas`, `areanum`.

### AAS_AllocTmpNode / AAS_FreeTmpNode
- **Signature:** `tmp_node_t *AAS_AllocTmpNode(void)` / `void AAS_FreeTmpNode(tmp_node_t *)`
- **Purpose:** Pool-allocate nodes from `tmp_nodebuf_t` slabs (128 nodes each).
- **Side effects:** May allocate a new `tmp_nodebuf_t` slab; modifies `tmpaasworld.nodebuffer`, `numnodes`.
- **Notes:** `AAS_FreeTmpNode` only decrements the counter; memory is freed in bulk via `AAS_FreeTmpAAS`.

### AAS_GapFace
- **Signature:** `int AAS_GapFace(tmp_face_t *tmpface, int side)`
- **Purpose:** Returns true if a face represents a gap (a drop-off) as seen from a given side.
- **Inputs:** Face and side index (0=front, 1=back)
- **Outputs/Return:** Non-zero if gap, 0 otherwise.
- **Calls:** `DotProduct`, `VectorCopy`, `VectorInverse`
- **Notes:** Excludes faces already flagged `FACE_GROUND | FACE_SOLID`.

### AAS_GroundFace
- **Signature:** `int AAS_GroundFace(tmp_face_t *tmpface)`
- **Purpose:** Returns true if the face normal is shallow enough relative to inverse gravity to stand on.
- **Inputs:** Face (must have `FACE_SOLID` set)
- **Outputs/Return:** Non-zero if ground, 0 otherwise.

### AAS_AddFaceSideToArea / AAS_RemoveFaceFromArea
- **Signature:** `void AAS_AddFaceSideToArea(tmp_face_t *, int side, tmp_area_t *)` / `void AAS_RemoveFaceFromArea(tmp_face_t *, tmp_area_t *)`
- **Purpose:** Insert or unlink a face from an area's doubly-linked per-side face list.
- **Side effects:** Modifies `tmp_face_t::frontarea/backarea`, `tmp_face_t::prev/next`, `tmp_area_t::tmpfaces`.

### AAS_CreateArea
- **Signature:** `tmp_node_t *AAS_CreateArea(node_t *node)`
- **Purpose:** Converts a single BSP leaf node into a `tmp_area_t` by iterating its portals, allocating faces, classifying face flags, and returning a leaf `tmp_node_t`.
- **Inputs:** BSP leaf `node_t`
- **Outputs/Return:** Allocated `tmp_node_t` pointing to the new area.
- **Side effects:** Allocates `tmp_face_t`, `tmp_area_t`, `tmp_node_t`; sets `portal_t::tmpface`.
- **Calls:** `AAS_AllocTmpArea`, `AAS_AllocTmpFace`, `AAS_AllocTmpNode`, `AAS_AddFaceSideToArea`, `AAS_GroundFace`, `AAS_FlipAreaFaces`, `AAS_CheckArea`, `ReverseWinding`, `CopyWinding`

### AAS_CreateAreas_r
- **Signature:** `tmp_node_t *AAS_CreateAreas_r(node_t *node)`
- **Purpose:** Recursively traverses the BSP tree; creates interior `tmp_node_t` for branches and calls `AAS_CreateArea` for non-solid leaves. Solid leaves return NULL.
- **Calls:** `AAS_AllocTmpNode`, `AAS_CreateArea`, self (recursive)

### AAS_CreateAreas
- **Signature:** `void AAS_CreateAreas(node_t *node)`
- **Purpose:** Entry point for BSP→AAS area extraction; stores root in `tmpaasworld.nodes`.
- **Calls:** `AAS_CreateAreas_r`

### AAS_CreateAreaSettings
- **Signature:** `void AAS_CreateAreaSettings(void)`
- **Purpose:** Iterates all valid areas, aggregates face flags, and allocates/populates `tmp_areasettings_t` per area.
- **Side effects:** Allocates `tmp_areasettings_t` for each area via `GetClearedMemory`.

### AAS_Create
- **Signature:** `void AAS_Create(char *aasfile)`
- **Purpose:** Top-level pipeline: BSP construction → portalization → flood fill → AAS extraction → multi-pass merging/subdivision → file storage.
- **Inputs:** Output AAS filename.
- **Side effects:** Calls the full BSP and AAS pipeline; writes AAS file via `AAS_StoreFile`.
- **Calls:** `ProcessWorldBrushes`, `Tree_PruneNodes`, `MakeTreePortals`, `FloodEntities`, `FillOutside`, `LeakFile`, `AAS_InitTmpAAS`, `AAS_CreateAreas`, `Tree_Free`, `AAS_MergeAreaFaces`, `AAS_GravitationalSubdivision`, `AAS_MergeAreas`, `AAS_PruneNodes`, `AAS_FlipSharedFaces`, `AAS_MergeAreaPlaneFaces`, `AAS_LadderSubdivision`, `AAS_MeltAreaFaceWindings`, `AAS_RemoveTinyFaces`, `AAS_CreateAreaSettings`, `AAS_StoreFile`, `AAS_FreeTmpAAS`
- **Notes:** Checks `cancelconversion` at multiple points to allow early termination.

## Control Flow Notes
This file is part of the **offline map compilation** tool (BSPC), not the runtime engine. `AAS_Create` is the primary entry point called from `bspc.c` after a map is loaded. The pipeline is:

1. BSP tree construction (`ProcessWorldBrushes`)
2. Portal generation (`MakeTreePortals`)
3. Flood fill / leak detection (`FloodEntities`, `FillOutside`)
4. Recursive BSP→AAS conversion (`AAS_CreateAreas`)
5. Multi-pass geometry cleanup: face merging, gravitational subdivision, area merging, node pruning, shared-face flipping, ladder subdivision, edge melting
6. Settings generation (`AAS_CreateAreaSettings`)
7. Serialization (`AAS_StoreFile`)

## External Dependencies
- `qbsp.h` — BSP types (`node_t`, `portal_t`, `plane_t`, `tree_t`), map globals (`mapplanes`, `entities`, `cancelconversion`, `freetree`, `source`), BSP pipeline functions
- `aasfile.h` — AAS file format constants (`FACE_*`, `AREA_*`, `AREACONTENTS_*`, `PRESENCE_*`)
- `aas_create.h` — Declarations for `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmp_aas_t`
- `aas_store.h` — `AAS_StoreFile`
- `aas_gsubdiv.h` — `AAS_GravitationalSubdivision`, `AAS_LadderSubdivision`
- `aas_facemerging.h` — `AAS_MergeAreaFaces`, `AAS_MergeAreaPlaneFaces`
- `aas_areamerging.h` — `AAS_MergeAreas`
- `aas_edgemelting.h` — `AAS_MeltAreaFaceWindings`
- `aas_prunenodes.h` — `AAS_PruneNodes`
- `aas_cfg.h` — `cfg` (physics config: `phys_gravitydirection`, `phys_maxsteepness`, `allpresencetypes`)
- `surfaceflags.h` — BSP content flags (`CONTENTS_SOLID`, `CONTENTS_WATER`, `CONTENTS_LADDER`, etc.)
- **Defined elsewhere:** `GetClearedMemory`, `FreeMemory`, `FreeWinding`, `ReverseWinding`, `CopyWinding`, `WindingCenter`, `WindingPlane`, `WindingArea`, `RemoveColinearPoints`, `WindingError`, `Log_Print`, `Log_Write`, `qprintf`, `Error`, `I_FloatTime`, `ThreadSetDefault`, `DotProduct`, `VectorCopy`, `VectorInverse`, `VectorScale`, `VectorAdd`
