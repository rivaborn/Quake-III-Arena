# code/bspc/aas_store.c — Enhanced Analysis

## Architectural Role

This file is the final **serialization codegen** in the offline bot-navigation compiler pipeline. It bridges the intermediate (`tmp_aas_t`) representation—produced by topology/merge passes in `aas_create.c`—to the packed, deduplicated runtime (`aas_t`) singleton loaded by botlib at engine startup. As part of `code/bspc/` (a standalone tool), it is never linked into the game engine itself; it runs once during map compilation to produce `.aas` binary files consumed by `code/botlib/be_aas_file.c` at runtime.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/bspc/bspc.c`** — calls `AAS_StoreFile()` as the final step of offline AAS compilation; chains: `AAS_Create()` → `AAS_Optimize()` → `AAS_StoreFile()`.
- **`code/bspc/aas_file.c`** — reads the in-memory `aasworld` built here and serializes it via `AAS_WriteAASFile()`.
- **botlib at runtime** — `code/botlib/be_aas_file.c` deserializes the output binary back into `aasworld` and uses all structures defined here.

### Outgoing (what this file depends on)

- **`code/bspc/aas_create.h/c`** — source: `tmpaasworld` (temporary faces, areas, BSP tree) and allocation/free utilities.
- **`code/bspc/aas_cfg.h`** — `cfg` (bounding box configuration) copied into `aasworld.bboxes[]`.
- **`qbsp.h`** — `plane_t`, `winding_t`, `mapplanes[]`, `nummapplanes`; BSP primitives and math utilities.
- **`code/botlib/aasfile.h`** — struct definitions for all `aas_*_t` types; this file is the "producer" for botlib's "consumer."
- **Utility functions** — `GetClearedMemory`, `FreeMemory`, `Error`, `Q_rint`, `VectorCopy`, `PlaneTypeForNormal`, etc.

## Design Patterns & Rationale

### 1. **Hash-Table Deduplication via Integer Chaining**
Three independent hash tables (vertices, edges, planes) use **fixed-size tables with singly-linked chains**:
- Vertex: `VERTEX_HASH_SIZE² = ~8K buckets` (spatial grid hash); chain array is `aas_vertexchain[MAX_VERTEXES]`.
- Edge: `EDGE_HASH_SIZE = 1024` (simple modulo hash); chain via `aas_edgechain[MAX_EDGES]`.
- Plane: `PLANE_HASH_SIZE = 1024`; chain via `aas_planechain[MAX_PLANES]`.

**Rationale:** Pre-era of generalized hash tables; fixed allocation avoids dynamic resizing overhead. Integer chaining is cache-hostile but memory-efficient for compile tools. Spatial hashing for vertices ensures geometric proximity → hash proximity, reducing collisions.

### 2. **Epsilon-Snap Grid Quantization**
`AAS_GetVertex()` snaps coordinates to integers if within `INTEGRAL_EPSILON (0.01)`, reducing redundant near-identical vertices. This is a **lossy compression for runtime efficiency** while preserving BSP precision.

### 3. **Bidirectional Edge Normalization**
`AAS_FindHashedEdge()` stores edges in both directions (v1→v2 and v2→v1) but returns a **signed edge index**: positive for forward, negative for reversed. This halves storage while preserving direction information.

### 4. **Dual Plane Storage (Optional)**
When `STOREPLANESDOUBLE` is defined, both a plane and its negation are stored. This is a **runtime optimization trade-off**: O(2n) plane storage but O(1) plane-flip at face-time vs. O(n) storage but O(plane lookup) at face-time.

### 5. **Two-Pass Compilation Strategy**
- **Pass 1** (in `aas_create.c`): Build flexible `tmp_aas_t` with linked lists, merging, optimization.
- **Pass 2** (this file): Pack into contiguous `aas_t` arrays via bottom-up tree walk + deduplication.

**Why:** Temporary structures support arbitrary insertion/deletion; final arrays are cache-efficient and disk-serializable.

## Data Flow Through This File

```
Input:  tmpaasworld (linked-list faces/areas, BSP tree)
        cfg (bounding box array)
        
↓ AAS_StoreFile(filename)
  ├─ AAS_AllocMaxAAS()        [allocate all aasworld arrays sized from tmpaasworld counts]
  ├─ AAS_StoreBoundingBoxes() [copy cfg → aasworld.bboxes[]]
  └─ AAS_StoreTree_r()        [recursive tree walk]
       ├─ for each leaf → AAS_StoreArea()
       │    ├─ for each face → AAS_GetFace()
       │    │    ├─ AAS_GetPlane()   [intern plane; return index]
       │    │    └─ for each edge → AAS_GetEdge()
       │    │         ├─ AAS_GetVertex() × 2  [intern endpoints]
       │    │         └─ AAS_FindHashedEdge() [reuse or create edge]
       │    └─ append face/area index entries to aasworld
       └─ for each internal node → AAS_GetPlane()
       
Output: aasworld (all arrays populated)
        aasworld.loaded = true
```

**State transitions:**
- Vertex: hash-lookup → found (return old index) or create (append, update hash chain, return new index).
- Edge: similarly, but also check orientation and return signed index.
- Plane: check forward + backward (if `STOREPLANESDOUBLE`); flip axial planes to positive-normal-first order for BSP sign consistency.
- Face: concatenate all edge indices into a single global `edgeindex` array; store base offset and count in `aas_face_t`.
- Area: collect all incident faces, compute bounds and centroid, store in `aas_area_t` and `aas_areasettings_t`.

## Learning Notes

### Idiomatic to Q3 / Early 2000s Game Engine Design

1. **Manual Memory Management:** No abstraction layer; raw `GetClearedMemory` / `FreeMemory` with explicit loop initialization of hash tables to `-1`. Compare to modern engines (e.g., UE5) which use containers with automatic hashing.

2. **Offline Tool Architecture:** Complete separation between compile-time (`code/bspc/`) and runtime (`code/botlib/`). The compiler is a **standalone executable**, not linked into the engine. This is idiomatic for **map preprocessing** (Q3, Doom, Unreal), avoiding runtime overhead.

3. **Spatial Hash for Geometry:** Using a 2D grid hash for vertices (`x,y` grid bins) while ignoring `z`. This reflects the **layer-cake BSP design** where vertical separation is natural; horizontally-close vertices are likely to share edges.

4. **Contiguous Array Packing:** Final `aasworld` uses contiguous arrays, not trees or linked lists. This is **cache-optimal** for runtime queries (`AAS_PointAreaNum`, `AAS_AreaCluster`), critical for per-frame AI queries in tight loops.

5. **Epsilon Thresholds Tuned by Iteration:** Comments show evolution (e.g., `VERTEX_EPSILON` changed from 0.5 to 0.1, `DIST_EPSILON` from 0.9 to 0.05). This indicates **empirical tuning** on shipped maps.

### Modern Contrast

- **Modern engines** (Unity, Godot, UE5) use **runtime navigation mesh generation** (e.g., Recast) or precomputed graphs stored in scenes, not separate offline formats.
- **No JIT/specialization:** All hash functions are generic; no template metaprogramming or specialization for different hash key widths.
- **Single global singleton:** `aasworld` is a static global, not a handle/resource. Multi-level loading not supported (each map replaces the entire AAS world).

## Potential Issues

1. **No Hash Table Overflow Safeguard**
   - If `aasworld.numvertexes ≥ max_aas.max_vertexes`, `AAS_GetVertex()` calls `Error()` (fatal).
   - No adaptive resizing or LRU eviction; capacity is all-or-nothing.
   - **Risk:** Maps with degenerate geometry (e.g., highly subdivided terrain) could exceed bounds if max estimation in `AAS_InitMaxAAS()` is conservative.

2. **Out-of-Bounds Hash Index Not Checked at Insertion**
   - `AAS_GetVertex()` checks hash value `h == -1` on *lookup* but proceeds to *insert* without re-checking.
   - A vertex outside bounds will be appended to `aasworld.vertexes` but not hashed → **memory leak** + future lookup misses.
   - **Risk:** Stale vertices in the array, invisible to hash queries.

3. **Silent Epsilon Snapping**
   - Vertices within `INTEGRAL_EPSILON` of integer coordinates are silently snapped. No warning logged.
   - **Risk:** Subtle geometry alteration on maps with intentional sub-unit placement (e.g., precise item spawns).

4. **No Capacity Assertions on Hash Chain Updates**
   - After `AAS_AddEdgeToHash(edgenum)`, there's no check that `edgenum < aasworld.numedges`.
   - If called with stale index, overwrites hash chain memory.

5. **Duplicate Plane Storage (STOREPLANESDOUBLE)**
   - Adds 2× plane array size but doesn't fail gracefully if `max_aas.max_planes` is exceeded during dual-plane insertion.
   - **Risk:** Plane overflow mid-insertion could corrupt the array.

---

**Summary:** This file exemplifies **pre-modern offline geometry baking**: simple but rigid. The hash-table deduplication is effective for typical maps, and the epsilon quantization reduces data. However, the fixed-size allocation and lack of overflow recovery make it brittle on atypical geometry. For a shipped game with QA-verified maps, acceptable; for a production engine, would benefit from dynamic resizing, bounds checking, and per-insertion validation.
