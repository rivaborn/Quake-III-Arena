# code/bspc/aas_store.c

## File Purpose
Converts the intermediate (temporary) AAS representation built during BSP compilation into the final packed `aas_t` world structure, deduplicating vertices, edges, planes, and faces via hash tables before the data is serialized to disk.

## Core Responsibilities
- Allocate and free all `aasworld` arrays sized to worst-case maximums derived from the tmp world
- Deduplicate and intern vertices, edges, and planes into contiguous arrays using hash chains
- Convert `tmp_face_t` windings into `aas_face_t` records (with edge index)
- Convert `tmp_area_t` nodes into `aas_area_t` records with bounds and centroid
- Recursively walk `tmp_node_t` tree and emit `aas_node_t` BSP nodes
- Copy bounding box configuration from `cfg` into `aasworld`
- Log allocation / deallocation totals

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `max_aas_t` | struct | Per-lump capacity caps computed from the tmp world before allocation |
| `aas_t` (`aasworld`) | global struct (defined in botlib) | The live, fully-packed AAS world being built |
| `tmp_aas_t` (`tmpaasworld`) | global struct | Source temporary AAS from the creation pass |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aasworld` | `aas_t` | global | The single AAS world being populated |
| `max_aas` | `max_aas_t` | file-static (global) | Computed capacity limits for each array |
| `allocatedaasmem` | `int` | global | Running byte count of AAS heap allocated |
| `groundfacesonly` | `int` | global | Filter flag (currently unused/false) |
| `aas_vertexchain` | `int *` | global | Per-vertex next-chain for vertex hash table |
| `aas_hashverts` | `int[VERTEX_HASH_SIZE²]` | global | Vertex hash table heads |
| `aas_planechain` | `int *` | global | Per-plane next-chain for plane hash table |
| `aas_hashplanes` | `int[PLANE_HASH_SIZE]` | global | Plane hash table heads |
| `aas_edgechain` | `int *` | global | Per-edge next-chain for edge hash table |
| `aas_hashedges` | `int[EDGE_HASH_SIZE]` | global | Edge hash table heads |

## Key Functions / Methods

### AAS_InitMaxAAS
- **Signature:** `void AAS_InitMaxAAS(void)`
- **Purpose:** Counts tmp faces, winding points, and areas to derive upper-bound capacities for all `aasworld` arrays.
- **Inputs:** Reads `tmpaasworld.faces`, `tmpaasworld.areas`, `tmpaasworld.nodes`, `nummapplanes`.
- **Outputs/Return:** Fills `max_aas`.
- **Side effects:** None beyond writing `max_aas`.
- **Calls:** `AAS_CountTmpNodes`

### AAS_AllocMaxAAS
- **Signature:** `void AAS_AllocMaxAAS(void)`
- **Purpose:** Heap-allocates all `aasworld` arrays and hash-chain arrays; initialises all hash tables to `-1`.
- **Inputs:** `max_aas` (via `AAS_InitMaxAAS`).
- **Outputs/Return:** Populated `aasworld` pointers; updated `allocatedaasmem`.
- **Side effects:** Large heap allocations; prints memory total to log.
- **Calls:** `AAS_InitMaxAAS`, `GetClearedMemory`, `Log_Print`, `PrintMemorySize`

### AAS_FreeMaxAAS
- **Signature:** `void AAS_FreeMaxAAS(void)`
- **Purpose:** Frees every `aasworld` array and the three hash-chain arrays; resets all counts.
- **Side effects:** Deallocates heap; logs freed amount; zeroes `allocatedaasmem`.
- **Calls:** `FreeMemory`, `Log_Print`, `PrintMemorySize`

### AAS_GetVertex
- **Signature:** `qboolean AAS_GetVertex(vec3_t v, int *vnum)`
- **Purpose:** Snaps a 3-D point to integer grid when close, looks it up in the vertex hash, inserts it if new.
- **Inputs:** `v` – world-space point; `*vnum` – output index.
- **Outputs/Return:** `true` if already existed, `false` if newly inserted; `-1` in `*vnum` if out of bounds.
- **Side effects:** May append to `aasworld.vertexes` and update hash chains.
- **Calls:** `AAS_HashVec`, `Q_rint`, `fabs`, `VectorCopy`
- **Notes:** Falls back to linear scan when `VERTEX_HASHING` is disabled.

### AAS_GetEdge
- **Signature:** `qboolean AAS_GetEdge(vec3_t v1, vec3_t v2, int *edgenum)`
- **Purpose:** Interns an edge (v1→v2) into `aasworld.edges`; degenerate or out-of-bounds edges return `*edgenum = 0`.
- **Outputs/Return:** `true` if found, `false` if newly stored; negative `*edgenum` denotes reversed edge.
- **Side effects:** May append to `aasworld.edges`; updates edge hash.
- **Calls:** `AAS_GetVertex`, `AAS_FindHashedEdge`, `AAS_AddEdgeToHash`

### AAS_GetPlane
- **Signature:** `qboolean AAS_GetPlane(vec3_t normal, vec_t dist, int *planenum)`
- **Purpose:** Interns a plane (and, when `STOREPLANESDOUBLE`, its negation) into `aasworld.planes`.
- **Outputs/Return:** `true` if already existed, `false` if newly stored.
- **Side effects:** Appends 1 or 2 plane entries; updates plane hash; may flip axial planes to positive-normal-first order.
- **Calls:** `AAS_FindHashedPlane`, `AAS_AddPlaneToHash`, `PlaneTypeForNormal`, `VectorCopy`, `VectorNegate`

### AAS_GetFace
- **Signature:** `qboolean AAS_GetFace(winding_t *w, plane_t *p, int side, int *facenum)`
- **Purpose:** Builds an `aas_face_t` from a winding by interning its plane and all edges into the global edge index.
- **Inputs:** `w` – polygon; `p` – face plane; `side` – unused in current code path.
- **Outputs/Return:** `true` on success, `false` if face was degenerate (too few valid edges).
- **Side effects:** Appends to `aasworld.faces` and `aasworld.edgeindex`.
- **Calls:** `AAS_GetPlane`, `AAS_GetEdge`, `Log_Write`

### AAS_StoreArea
- **Signature:** `int AAS_StoreArea(tmp_area_t *tmparea)`
- **Purpose:** Converts one `tmp_area_t` (following merge chains) into an `aas_area_t`, building its face list, bounds, and centroid; also stores area settings.
- **Outputs/Return:** Negative area number on success, `0` if area had no faces.
- **Side effects:** Appends area, its face index entries, and area settings to `aasworld`; sets `tmparea->aasareanum`; prints progress.
- **Calls:** `AAS_GetFace`, `AAS_StoreAreaSettings`, `AddPointToBounds`, `VectorAdd`, `VectorScale`, `ClearBounds`, `ReverseWinding`, `FreeWinding`, `qprintf`

### AAS_StoreTree_r
- **Signature:** `int AAS_StoreTree_r(tmp_node_t *tmpnode)`
- **Purpose:** Recursively converts the tmp BSP tree into `aas_node_t` entries; leaf nodes dispatch to `AAS_StoreArea`.
- **Outputs/Return:** Node index (positive), area index (negative), or 0 for solid leaf.
- **Side effects:** Appends to `aasworld.nodes`; triggers all area/face/edge/vertex storage.
- **Calls:** `AAS_StoreArea`, `AAS_GetPlane`

### AAS_StoreFile
- **Signature:** `void AAS_StoreFile(char *filename)`
- **Purpose:** Top-level entry point: allocates storage, copies bounding boxes, drives the tree walk, marks the world loaded.
- **Side effects:** Full `aasworld` population; sets `aasworld.loaded = true`.
- **Calls:** `AAS_AllocMaxAAS`, `AAS_StoreBoundingBoxes`, `AAS_StoreTree_r`, `Log_Write`, `qprintf`

## Control Flow Notes
This file is a **compile-time / offline** tool (BSPC), not part of the game runtime. `AAS_StoreFile` is called once after the BSP-to-AAS conversion pass completes. The result is an in-memory `aasworld` ready to be serialized by `AAS_WriteAASFile` (declared in `aas_file.h`). There is no per-frame update loop.

## External Dependencies
- `qbsp.h` — `plane_t`, `winding_t`, `mapplanes`, `nummapplanes`, math utilities
- `botlib/aasfile.h` — all `aas_*_t` struct definitions, face/area flag constants
- `aas_create.h` — `tmp_face_t`, `tmp_area_t`, `tmp_node_t`, `tmpaasworld`
- `aas_cfg.h` — `cfg` (bounding box config)
- **Defined elsewhere:** `GetClearedMemory`, `FreeMemory`, `Log_Print`, `Log_Write`, `PrintMemorySize`, `qprintf`, `Error`, `Q_rint`, `VectorCopy/Add/Scale/Negate/Clear`, `AddPointToBounds`, `ClearBounds`, `ReverseWinding`, `FreeWinding`, `PlaneTypeForNormal`
