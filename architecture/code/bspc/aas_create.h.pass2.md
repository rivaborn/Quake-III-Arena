# code/bspc/aas_create.h — Enhanced Analysis

## Architectural Role

This header file is part of the **offline BSP-to-AAS compiler (bspc)**, which operates entirely outside the runtime engine. It defines the intermediate data structures and function contracts for the temporary AAS world created during the map compilation pipeline. Specifically, it bridges the gap between raw BSP geometry and the finalized navigation graph: the `tmp_*` structures here are mutable working representations that the compilation pipeline (area merging, face merging, gravitational subdivision, reachability analysis) transforms before serialization via `aas_store.c`. This clean separation keeps the runtime botlib library small and immutable—botlib never touches these structures; it only consumes the final `.aas` binary format.

## Key Cross-References

### Incoming (who depends on this file)

The temporary AAS pipeline is consumed exclusively by **code/bspc/** subsystem modules:

- **`aas_create.c`**: Defines the core allocation/deallocation and linkage helpers (`AAS_AllocTmp{Face,Area,Node}`, `AAS_Add/RemoveFaceSideToArea`, `AAS_FlipAreaFaces`, `AAS_GapFace`, `AAS_GroundFace`, `AAS_CheckArea`)
- **`aas_areamerging.c`**: Consumes `tmp_area_t` and mutates the `mergedarea` pointer to consolidate adjacent areas
- **`aas_facemerging.c`, `aas_edgemelting.c`**: Optimize face geometry during area construction
- **`aas_gsubdiv.c`**: Performs gravitational subdivision (e.g., ladders) by mutating `tmp_area_t` and `tmp_face_t` linked lists
- **`aas_prunenodes.c`**: Trims unreachable nodes from `tmp_node_t` tree
- **`be_aas_bspc.c`**: Performs reachability analysis and cluster computation on the temporary structure; serves as the pipeline orchestrator
- **`bspc.c`**: Top-level driver that calls `AAS_Create` to initiate compilation

### Outgoing (what this file depends on)

- **`winding_t`** (from `l_poly.h` / `qbsp.h`): Polygon representation used in `tmp_face_t::winding`
- **No dependencies on botlib proper**: Unlike the runtime botlib, this offline tool does not import botlib symbols; instead, **botlib reuses parts of the bspc AAS code** (cluster, reach, optimize modules) via a compatibility shim (`be_aas_bspc.c` exports botlib-compatible signatures)
- **Implicit BSP state**: Functions assume a loaded BSP world available globally (not declared here)

## Design Patterns & Rationale

### 1. **Intermediate Representation (Temporary Structures)**
All structures are prefixed `tmp_*` to signal they exist only during compilation. This is a classical compiler pattern: work with highly mutable intermediate forms (doubly-linked lists, reference pointers, flags) before lowering to the immutable runtime format. The final `.aas` binary uses dense arrays and indices, not linked lists.

### 2. **Doubly-Linked Lists with Dual Semantics**
`tmp_face_t` uses both `prev[2]/next[2]` (per-side area linkage) and `l_prev/l_next` (global face list). This dual linking allows:
- Fast removal of a face from one area without traversing the global list
- Iteration over all faces for sanity checks and serialization

### 3. **Merge Tracking via Pointer Indirection**
The `tmp_area_t::mergedarea` field captures the result of area consolidation. When two areas are merged, the old area points to the new one, allowing the entire graph to be updated by following this single pointer. This avoids rebuilding adjacency lists.

### 4. **Slab Allocator (Fixed-Size Buffers)**
`tmp_nodebuf_t` chains `NODEBUF_SIZE` (128) nodes per slab, avoiding per-allocation overhead and fragmentation. This is practical for offline tools where allocation patterns are predictable. The global `tmpaasworld.nodebuffer` chain is a simple linked list of slabs.

### 5. **Separation of Geometry from Metadata**
`tmp_area_t` holds geometry (faces, presence type) while `tmp_areasettings_t` holds metadata (contents flags, model number, reachability). This allows the compiler to compute and validate settings independently.

## Data Flow Through This File

**Input → Transform → Output:**

1. **BSP Geometry (Input)**  
   Loaded into global BSP structures elsewhere; accessed by functions to extract planes, windings, entity keys.

2. **Area/Face/Node Creation** (`AAS_Create`)  
   - Traverse BSP tree recursively
   - For each leaf, allocate `tmp_area_t` (via `AAS_AllocTmpArea`)
   - For each plane/surface, allocate `tmp_face_t` (via `AAS_AllocTmpFace`)
   - Link faces into area via `AAS_AddFaceSideToArea` (updates `prev[side]`/`next[side]` and `frontarea`/`backarea`)
   - Allocate `tmp_node_t` (via `AAS_AllocTmpNode`; carved from slab buffer)

3. **Geometry Validation & Normalization**  
   - `AAS_FlipAreaFaces`: Corrects face winding orientation
   - `AAS_CheckArea`: Validates geometric consistency (e.g., no missing faces, valid topology)
   - `AAS_GapFace` / `AAS_GroundFace`: Classify surfaces (used during reachability analysis)

4. **Compilation Passes** (external to this header, but consume these structures)
   - **Area Merging**: Combine coplanar adjacent areas; update `tmp_area_t::mergedarea`
   - **Face/Edge Merging**: Consolidate coplanar faces; mutate `tmp_face_t` windings and links
   - **Gravitational Subdivision**: Insert new faces/areas for ledges, ladders; restructure linked lists
   - **Node Pruning**: Remove redundant nodes from `tmp_node_t` tree
   - **Reachability**: Compute inter-area reachability; populate `tmp_areasettings_t::numreachableareas`

5. **Serialization & Output** (via `aas_store.c`)  
   - Convert linked-list graph to dense arrays (plane indices, edge indices, vertex indices)
   - Write `.aas` binary file with header, lumps (vertices, edges, faces, areas, reachability)

## Learning Notes

### Compilation as a Distinct Phase
Quake III separates **offline map compilation** (bspc) from **runtime game execution** (botlib). This design:
- **Reduces runtime complexity**: botlib has no allocation logic, no temporary structures, no graph-building overhead
- **Enables offline optimization**: expensive algorithms (area merging, reachability analysis) run once per map, not per bot-load
- **Maintains strict separation**: the runtime bot AI never allocates or modifies AAS; it only queries and caches

### Idiomatic Offline Tool Patterns
This file exemplifies offline-compiler idioms rarely seen in modern game engines:
- **Doubly-linked lists for graph mutation**: Modern engines use indices or handle-based ECS; offline tools often prefer pointer-based graphs for ease of structural manipulation
- **Simple slab allocators**: Contrast with arena/bump allocators or generational GC in runtime engines; slabs are predictable and fragmenting offline is acceptable
- **Global singleton state** (`tmpaasworld`): Not thread-safe; acceptable offline because the compiler is single-threaded
- **In-place graph transformations**: Modern engines prefer immutable intermediate representations (SSA-like); this tool mutates structures in-place

### Game Engine Concepts
This file demonstrates:
- **Scene graph variant**: The `tmp_node_t` tree mirrors the BSP's spatial hierarchy; at runtime, botlib traverses this to answer "which area is this point in?" queries
- **Graph compaction**: Area merging is similar to graph node coalescing in static analysis; the `mergedarea` pointer is a forward reference used during transitive closure
- **Reachability as a compile-time artifact**: Unlike dynamic pathfinding at runtime, reachability is precomputed and serialized; this is cost-shifting from frame-time to compile-time

## Potential Issues

1. **Typo in GPL header**: Line 18 mentions "Foobar" instead of the project name; this is upstream boilerplate from the original Q3A source release but should be corrected for clarity.

2. **No null-check documentation**: The public function declarations do not document null-pointer expectations (e.g., is `tmpface` ever NULL?). The implementation in `aas_create.c` likely has defensive checks, but callers cannot know from this header.

3. **Fixed slab size (`NODEBUF_SIZE=128`)**: If a map requires more than 128 nodes before the first buffer fills, a new slab is allocated. However, if allocation patterns are pathological, this could fragment memory or hide bugs. No safeguards against unbounded growth visible here.

4. **Merge pointer assumes single consolidation**: The `mergedarea` field assumes each area is merged at most once. If the pipeline permits re-merging (which is unlikely but not prohibited by the data structure), transitive closure would require following chains, risking cycles or traversal bugs.
