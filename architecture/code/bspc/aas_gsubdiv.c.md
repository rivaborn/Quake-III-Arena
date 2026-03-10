# code/bspc/aas_gsubdiv.c

## File Purpose
Implements gravitational and ladder-based geometric subdivision of temporary AAS areas during the BSPC map compilation process. It splits AAS areas along planes to ensure areas do not contain mixed ground/gap or ground/ladder regions that would confuse bot navigation.

## Core Responsibilities
- Split individual `tmp_face_t` polygons along a plane, producing front/back fragments
- Construct a split winding that clips the splitting plane to the convex bounds of an area
- Evaluate candidate split planes for quality (minimizing face splits, avoiding epsilon slivers)
- Find the best vertical split plane between ground/gap face pairs in an area
- Recursively subdivide the AAS BSP tree via gravitational subdivision (ground vs. gap separation)
- Recursively subdivide areas containing both ladder faces and ground faces via a horizontal plane through the lowest ladder vertex
- Patch the global BSP tree after ladder subdivisions to keep it consistent

## Key Types / Data Structures
None defined locally; uses types from included headers.

| Name | Kind | Purpose |
|------|------|---------|
| `tmp_face_t` | struct (typedef) | Temporary AAS face with winding, plane, flags, and area links |
| `tmp_area_t` | struct (typedef) | Temporary AAS area owning a linked list of faces |
| `tmp_node_t` | struct (typedef) | BSP tree node referencing either a child pair or a `tmp_area_t` leaf |
| `plane_t` | struct (typedef) | Map plane with normal, dist, type, signbits |
| `winding_t` | struct (typedef) | Convex polygon (defined in poly library) |
| `cfg_t` | struct (typedef) | AAS configuration including gravity direction |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numgravitationalsubdivisions` | `int` | global | Running count of gravitational subdivisions performed; printed to console |
| `numladdersubdivisions` | `int` | global | Running count of ladder subdivisions performed; printed to console |

## Key Functions / Methods

### AAS_SplitFace
- **Signature:** `void AAS_SplitFace(tmp_face_t *face, vec3_t normal, float dist, tmp_face_t **frontface, tmp_face_t **backface)`
- **Purpose:** Clips a face's winding against the given plane, allocating new `tmp_face_t` structs for the front and back fragments.
- **Inputs:** Source face, split plane (normal + dist), output pointers.
- **Outputs/Return:** Writes to `*frontface` / `*backface`; either may be NULL if fully on one side.
- **Side effects:** Allocates new `tmp_face_t` via `AAS_AllocTmpFace`; original face is left unchanged (caller must free).
- **Calls:** `ClipWindingEpsilon`, `WindingIsTiny` (DEBUG only), `FreeWinding` (DEBUG only), `AAS_AllocTmpFace`, `Log_Write` (DEBUG).
- **Notes:** FACECLIP_EPSILON = 0.2 is used as clip tolerance. Tiny windings are discarded only in DEBUG builds.

### AAS_SplitWinding
- **Signature:** `winding_t *AAS_SplitWinding(tmp_area_t *tmparea, int planenum)`
- **Purpose:** Constructs the winding of intersection between a splitting plane and a convex area by starting with a base winding and chopping it against every face plane of the area.
- **Inputs:** Area to test against, plane number of the splitter.
- **Outputs/Return:** Heap-allocated `winding_t`, or NULL if the plane does not intersect the area.
- **Side effects:** Allocates a winding (caller must free).
- **Calls:** `BaseWindingForPlane`, `ChopWindingInPlace`.
- **Notes:** The side of each face used for chopping is computed from whether `face->frontarea == tmparea`.

### AAS_TestSplitPlane
- **Signature:** `int AAS_TestSplitPlane(tmp_area_t *tmparea, vec3_t normal, float dist, int *facesplits, int *groundsplits, int *epsilonfaces)`
- **Purpose:** Evaluates a candidate split plane by counting how many faces it splits and how many produce epsilon-thin slivers.
- **Inputs:** Area, candidate plane, output counters.
- **Outputs/Return:** `true` if plane is usable; outputs filled with counts.
- **Side effects:** Calls `AAS_SplitWinding` (allocs + frees a winding); calls `Log_Print` if the split plane coincides with a face plane.
- **Calls:** `FindFloatPlane`, `AAS_SplitWinding`, `FreeWinding`, `DotProduct`, `Log_Print`.
- **Notes:** Epsilon face detection range: `(FACECLIP_EPSILON, FACE_EPSILON)` = (0.2, 1.0).

### AAS_SplitArea
- **Signature:** `void AAS_SplitArea(tmp_area_t *tmparea, int planenum, tmp_area_t **frontarea, tmp_area_t **backarea)`
- **Purpose:** Core area split: creates a split face for the dividing plane, allocates two new areas, redistributes all original faces to the appropriate new area, marks the original area invalid.
- **Inputs:** Area to split, plane number, output area pointers.
- **Outputs/Return:** Writes new areas to `*frontarea` / `*backarea`.
- **Side effects:** Allocates faces and areas; frees original faces via `AAS_FreeTmpFace`; marks `tmparea->invalid = true`; calls validation/flip helpers.
- **Calls:** `AAS_FlipAreaFaces`, `AAS_CheckArea`, `AAS_SplitWinding`, `AAS_AllocTmpFace`, `AAS_AllocTmpArea`, `AAS_AddFaceSideToArea`, `AAS_RemoveFaceFromArea`, `AAS_SplitFace`, `AAS_FreeTmpFace`, `FreeWinding`, `Error`, `Log_Print`.
- **Notes:** Must be called before area merging for gravitational subdivision (BSP tree not refreshed here).

### AAS_FindBestAreaSplitPlane
- **Signature:** `int AAS_FindBestAreaSplitPlane(tmp_area_t *tmparea, vec3_t normal, float *dist)`
- **Purpose:** Searches all pairs of ground/gap faces in an area for a vertical separating plane, scoring each by `100 - facesplits - 2*groundsplits - 1000*epsilonfaces`.
- **Inputs:** Area to examine, output normal and dist for the best plane found.
- **Outputs/Return:** Returns non-zero if a suitable splitter was found.
- **Side effects:** Calls `Log_Write` for tiny windings and epsilon faces.
- **Calls:** `VectorCopy`, `VectorInverse`, `WindingIsTiny`, `AAS_GapFace`, `FindPlaneSeperatingWindings`, `AAS_TestSplitPlane`, `Log_Write`, `Log_Print` (AW_DEBUG only).
- **Notes:** Uses inverted gravity direction as the "vertical" axis. Only considers ground↔gap pairs.

### AAS_SubdivideArea_r
- **Signature:** `tmp_node_t *AAS_SubdivideArea_r(tmp_node_t *tmpnode)`
- **Purpose:** Recursively splits an area node until no more valid split planes are found.
- **Inputs:** BSP leaf node containing an area.
- **Outputs/Return:** The (possibly expanded) subtree rooted at this node.
- **Side effects:** Increments `numgravitationalsubdivisions`; allocates nodes and areas; modifies the BSP tree in place.
- **Calls:** `AAS_FindBestAreaSplitPlane`, `FindFloatPlane`, `AAS_SplitArea`, `AAS_AllocTmpNode`, `qprintf`.

### AAS_GravitationalSubdivision_r / AAS_GravitationalSubdivision
- **Signature:** `tmp_node_t *AAS_GravitationalSubdivision_r(tmp_node_t *)` / `void AAS_GravitationalSubdivision(void)`
- **Purpose:** Walk the entire BSP tree and call `AAS_SubdivideArea_r` on every area leaf.
- **Side effects:** Logs count; resets `numgravitationalsubdivisions`; modifies global `tmpaasworld.nodes` tree.
- **Calls:** `AAS_GravitationalSubdivision_r` (recursive), `AAS_SubdivideArea_r`, `Log_Write`, `qprintf`.

### AAS_LadderSubdivideArea_r
- **Signature:** `tmp_node_t *AAS_LadderSubdivideArea_r(tmp_node_t *tmpnode)`
- **Purpose:** If an area has both ladder and ground faces, finds the lowest ladder vertex and splits the area with a horizontal (Z-up) plane at that height, then refreshes the global BSP tree to replace the old area node.
- **Inputs:** BSP leaf node containing an area.
- **Outputs/Return:** Possibly expanded subtree.
- **Side effects:** Increments `numladdersubdivisions`; modifies `tmpaasworld.nodes` via `AAS_RefreshLadderSubdividedTree_r`.
- **Calls:** `AAS_SplitWinding`, `FreeWinding`, `AAS_SplitArea`, `AAS_AllocTmpNode`, `AAS_RefreshLadderSubdividedTree_r`, `FindFloatPlane`, `DotProduct`, `qprintf`.
- **Notes:** Skips liquid areas and non-normal presence areas. Skips if lowest ladder point is already on a ground plane (dist within ±1).

### AAS_RefreshLadderSubdividedTree_r
- **Signature:** `tmp_node_t *AAS_RefreshLadderSubdividedTree_r(tmp_node_t *tmpnode, tmp_area_t *tmparea, tmp_node_t *tmpnode1, tmp_node_t *tmpnode2, int planenum)`
- **Purpose:** Traverses the global BSP tree to find and replace the node referencing `tmparea` with the two new child nodes created by a ladder subdivision.
- **Notes:** Necessary because ladder subdivision is done post-merge and the BSP tree is not automatically updated.

## Control Flow Notes
- **Build-time only:** This file is part of the BSPC offline compiler, not the game runtime.
- `AAS_GravitationalSubdivision` is called **before** area merging; `AAS_LadderSubdivision` is called **after** face/edge merging.
- Both entry points walk `tmpaasworld.nodes` top-down, recursing into BSP children and subdividing area leaves in place.
- Gravitational subdivision uses the BSP naturally (no tree refresh needed); ladder subdivision requires explicit tree patching via `AAS_RefreshLadderSubdividedTree_r`.

## External Dependencies
- `qbsp.h` — `plane_t`, `mapplanes[]`, `FindFloatPlane`, `WindingIsTiny`, winding/polygon primitives
- `aasfile.h` — face flags (`FACE_GROUND`, `FACE_LADDER`, `FACE_GAP`), area content flags, presence types
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmpaasworld`, alloc/free/check helpers, `AAS_GapFace`
- `aas_store.h` — indirectly pulls in `aas_t aasworld`
- `aas_cfg.h` — `cfg` global (gravity direction via `cfg.phys_gravitydirection`)
- `FindPlaneSeperatingWindings` — defined elsewhere (polygon/geometry library)
- `ClipWindingEpsilon`, `BaseWindingForPlane`, `ChopWindingInPlace`, `FreeWinding` — winding library (defined elsewhere)
- `Log_Write`, `Log_Print`, `qprintf`, `Error` — logging/error utilities defined elsewhere
