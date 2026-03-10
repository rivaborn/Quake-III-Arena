# code/botlib/be_aas_optimize.c

## File Purpose
Post-processes the AAS (Area Awareness System) world data after reachability calculation by stripping all geometric data (vertices, edges, faces) except those marked with `FACE_LADDER`. This reduces the in-memory and on-disk AAS footprint to only the geometry still needed at runtime.

## Core Responsibilities
- Allocate parallel optimized arrays for all AAS geometric primitives
- Selectively retain only ladder faces (and their referenced edges/vertices)
- Remap old indices to new compacted indices via lookup tables
- Preserve sign conventions (face/edge side/direction) throughout remapping
- Patch reachability records to reference the new compacted indices
- Replace `aasworld` geometry arrays with the compacted versions and free the old ones

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `optimized_t` | struct | Transient workspace holding new compacted arrays and old→new index remapping tables for vertices, edges, and faces |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aasworld` | external (defined in `be_aas_def.h`) | global | The live AAS world being optimized in-place |
| `botimport` | external | global | Engine import table; used here only for `Print` after optimization |

## Key Functions / Methods

### AAS_KeepEdge
- **Signature:** `int AAS_KeepEdge(aas_edge_t *edge)`
- **Purpose:** Filter predicate for edges; currently always returns 1 (keep all edges that reach this point via face filtering).
- **Inputs:** `edge` — pointer to source edge (unused).
- **Outputs/Return:** Always `1`.
- **Side effects:** None.
- **Notes:** Stub; actual pruning is done at the face level by `AAS_KeepFace`. Comment says "KeepFace" — apparent copy-paste error in the original source.

### AAS_KeepFace
- **Signature:** `int AAS_KeepFace(aas_face_t *face)`
- **Purpose:** Filter predicate; retains a face only if it has the `FACE_LADDER` flag set.
- **Inputs:** `face` — pointer to source face.
- **Outputs/Return:** `1` if ladder face, `0` otherwise.
- **Side effects:** None.

### AAS_OptimizeEdge
- **Signature:** `int AAS_OptimizeEdge(optimized_t *optimized, int edgenum)`
- **Purpose:** Copies an edge (and its two vertices) into the optimized arrays if not already copied; returns the new index with original sign.
- **Inputs:** `optimized` — workspace; `edgenum` — signed source edge index (sign encodes direction).
- **Outputs/Return:** New signed edge index, or `0` if edge is filtered out.
- **Side effects:** Appends to `optimized->edges`, `optimized->vertexes`; updates `vertexoptimizeindex`, `edgeoptimizeindex`; increments `numedges`, `numvertexes`.
- **Calls:** `AAS_KeepEdge`, `VectorCopy`.

### AAS_OptimizeFace
- **Signature:** `int AAS_OptimizeFace(optimized_t *optimized, int facenum)`
- **Purpose:** Copies a face into the optimized arrays if it passes `AAS_KeepFace`; recursively optimizes all its edges.
- **Inputs:** `optimized` — workspace; `facenum` — signed source face index.
- **Outputs/Return:** New signed face index, or `0` if filtered.
- **Side effects:** Appends to `optimized->faces`, `optimized->edgeindex`; updates `faceoptimizeindex`; increments `numfaces`, `edgeindexsize`.
- **Calls:** `AAS_KeepFace`, `Com_Memcpy`, `AAS_OptimizeEdge`.

### AAS_OptimizeArea
- **Signature:** `void AAS_OptimizeArea(optimized_t *optimized, int areanum)`
- **Purpose:** Rebuilds one area's face list in the optimized arrays, dropping faces that don't survive `AAS_OptimizeFace`.
- **Inputs:** `optimized` — workspace; `areanum` — 1-based area index.
- **Side effects:** Writes `optimized->areas[areanum]`; appends to `optimized->faceindex`; updates `faceindexsize`.
- **Calls:** `Com_Memcpy`, `AAS_OptimizeFace`.

### AAS_OptimizeAlloc
- **Signature:** `void AAS_OptimizeAlloc(optimized_t *optimized)`
- **Purpose:** Allocates all compacted arrays and remapping index tables, sized to current `aasworld` counts (upper bounds).
- **Side effects:** Heap allocations via `GetClearedMemory`; sets initial dummy slot counts (`numedges = 1`, `numfaces = 1`) to reserve index 0 as invalid.
- **Calls:** `GetClearedMemory`.

### AAS_OptimizeStore
- **Signature:** `void AAS_OptimizeStore(optimized_t *optimized)`
- **Purpose:** Swaps `aasworld` geometry pointers to the compacted arrays, freeing the old ones, then frees the temporary index tables.
- **Side effects:** `FreeMemory` on old `aasworld` geometry arrays; updates all `aasworld` geometry fields; frees three `optimized` index arrays.
- **Calls:** `FreeMemory`.

### AAS_Optimize
- **Signature:** `void AAS_Optimize(void)`
- **Purpose:** Top-level entry point; orchestrates full AAS geometry compaction pass.
- **Side effects:** Rewrites `aasworld` geometry in-place; patches `aasworld.reachability[]` face/edge indices; calls `botimport.Print`.
- **Calls:** `AAS_OptimizeAlloc`, `AAS_OptimizeArea`, `AAS_OptimizeStore`, `botimport.Print`.
- **Notes:** Skips reachability index fixup for `TRAVEL_ELEVATOR`, `TRAVEL_JUMPPAD`, and `TRAVEL_FUNCBOB` — those travel types encode non-geometric data in `facenum`/`edgenum`.

## Control Flow Notes
Called once at AAS load time, after reachability computation, before the bot AI begins querying the world. Not part of the per-frame loop. Execution order: `AAS_OptimizeAlloc` → iterate areas (`AAS_OptimizeArea`) → patch reachabilities → `AAS_OptimizeStore`.

## External Dependencies
- `q_shared.h` — `VectorCopy`, `Com_Memcpy`, base types
- `l_memory.h` — `GetClearedMemory`, `FreeMemory`
- `aasfile.h` — `aas_vertex_t`, `aas_edge_t`, `aas_face_t`, `aas_area_t`, `aas_edgeindex_t`, `aas_faceindex_t`, `FACE_LADDER`
- `be_aas_def.h` — `aasworld` global (defined elsewhere), `aas_reachability_t`, `TRAVEL_ELEVATOR`, `TRAVEL_JUMPPAD`, `TRAVEL_FUNCBOB`, `TRAVELTYPE_MASK`
- `be_interface.h` — `botimport` (defined elsewhere)
