# code/bspc/faces.c — Enhanced Analysis

## Architectural Role
This file is a critical stage in the BSPC offline compilation pipeline (executed **one-time at map authoring, not at runtime**), immediately following BSP tree construction and portalization. It transforms raw portal windings into properly deduplicated, T-junction-free, merged, and subdivided BSP faces ready for final edge emission. Notably, `faces.c` operates entirely within the **offline tool** (`code/bspc/`) and has **zero runtime dependencies**—the runtime engine never calls these functions.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/writebsp.c`**: calls `GetEdge2()` (once per face vertex pair) during final `.bsp` file emission
- **`code/bspc/be_aas_bspc.c`** (AAS compiler): may share vertex dedup logic when building navigation geometry
- **`code/bspc/bspc.c`**: orchestrates the overall compilation pipeline; calls `MakeFaces()` and `FixTjuncs()` in sequence after BSP tree is complete

### Outgoing (what this file depends on)
- **Global BSP data** (from `bspc.c` / map globals):
  - `dvertexes[]`, `numvertexes`: output vertex array
  - `dedges[]`, `numedges`: output edge array
  - `texinfo[]`: surface properties (shader, flags, axes)
  - Flags: `noweld`, `notjunc`, `nomerge`, `nosubdiv`, `noshare`, `subdivide_size`
- **Winding/poly library** (external): `TryMergeWinding()`, `ClipWindingEpsilon()`, `CopyWinding()`, `ReverseWinding()`, `FreeWinding()`
- **Utility layer**: `qprintf()` (diagnostics), `Error()` (fatal errors), `Q_rint()` (rounding)

## Design Patterns & Rationale

**Spatial Hashing for Vertex Dedup**: `HashVec()` maps XY coordinates to 64×64 buckets; `GetVertexnum()` chains lookups within buckets. This is **not LRU or generic**: it exploits the assumption that map geometry spreads in XY and that Z variation within a bucket is rare. The linear fallback (disabled by `#define USE_HASHING`) exists for porting/debugging only.

**Global Scratch Buffers**: `superverts[]`, `edge_verts[]` are reused across recursive calls to avoid allocation overhead—a micro-optimization idiom from 1990s game engines. Modern code would use stack-allocated or heap-allocated temporary structures.

**Reentrancy via Global State**: `TestEdge()` recursively breaks edges at T-junctions using global `edge_start`, `edge_dir`, `edge_len`. The pattern is deterministic (single-threaded, predictable recursion depth), but prevents parallelization.

**Face Merging as a Post-Pass**: Rather than detecting merge opportunities during construction, merging is deferred to `MergeNodeFaces()`, allowing the algorithm to be independent of tree traversal order.

**Subdivide-by-Axes Heuristic**: `SubdivideFace()` chops faces along X and Y axes until all pieces fit within `subdivide_size` (default 512 or 256). This is a **texture-cache locality optimization** (reduces thrashing during software rasterization in older renderers) and is disabled in modern builds via `nosubdiv`.

## Data Flow Through This File

```
BSP Tree + Portals
      ↓
MakeFaces_r()  [creates face_t from each visible portal]
      ↓
MergeNodeFaces()  [merges coplanar same-texinfo faces on same node]
      ↓
SubdivideNodeFaces()  [splits large faces to fit cache]
      ↓
FixTjuncs()  [top-level: emit all verts, then fix edges]
  ├─→ EmitVertexes_r()  [recurse tree, deduplicate + snap vertices]
  ├─→ hashverts[], vertexchain[] populated  [hash table for dedup]
  └─→ FixEdges_r()  [recurse tree, call FixFaceEdges per face]
      └─→ TestEdge()  [recursively split edges at intermediate verts]
            └─→ superverts[] accumulated  [final vertex list for face]
      ↓
writebsp.c calls GetEdge2()  [emit dedges[], populate edgefaces[][] ]
```

**Key insight**: Face construction happens in two phases—geometry (MakeFaces, Merge, Subdivide) then vertex finalization (FixTjuncs)—allowing deferred vertex snapping and T-junction fixing without disrupting face topology.

## Learning Notes

**Idiomatic to This Era**:
- Heavy reliance on global state and file-scoped arrays (C static storage)
- Fixed-size limits and overflow counters (c_faceoverflows, c_tjunctions) as diagnostic instrumentation
- Conditional compilation (`#ifdef USE_HASHING`) for algorithm selection, not configuration
- Assume single-threaded, batch processing (not interactive or real-time)

**Modern Equivalents**:
- **ECS/Scene Graph**: Face data would be an entity type with component storage; merge/subdivide as systems
- **Functional / Immutable**: Each operation would return a new face tree; no in-place mutation
- **Spatial Indexing**: KD-tree or BVH instead of a flat 2D hash over XY
- **Lazy Evaluation**: Subdivide and merge on-demand during final export, not as separate passes

**Connection to Engine Concepts**:
- **BSP Face** ← analogue to modern GPU mesh / draw-call grouping (minimizes state changes)
- **T-Junction Fixing** ← critical for hardware rasterization correctness (prevents cracks in silhouettes)
- **Vertex Welding** ← modern GPU mesh optimization (index buffer deduplication)
- **Subdivide-by-Size** ← precursor to modern LOD and draw-call batching

## Potential Issues

1. **Fixed-Size Overflows**: `numsuperverts >= MAX_SUPERVERTS` and `faceoverflows` counter indicate that degenerate maps can cause early termination. No graceful fallback—just `Error()`.

2. **O(n²) T-Junction Detection**: `TestEdge()` iterates all `num_edge_verts` candidates per edge. Maps with many vertices near edge midpoints can be slow. No spatial acceleration for the inner loop.

3. **Global Vertex Hash Not Cleared Between Models**: `firstmodeledge` and `firstmodelface` track multi-model offsets, but hash tables (`hashverts[]`, `vertexchain[]`) are only cleared once at `FixTjuncs()` start, not per-model. Incorrect if called multiple times on different models in one run.

4. **Merging Doesn't Recompute Vertex Normals**: After face merging, the merged face's winding normals are recomputed via `TryMergeWinding()`, but there's no explicit re-validation that the merged result is still coplanar. Edge cases near floating-point boundaries could produce degenerate merged faces.

5. **Memory Leaks on Early Error**: If `Error()` is called during `MakeFaces()` or `FixTjuncs()`, in-flight allocated `face_t` objects (via `NewFaceFromFace()`, which calls `GetMemory()`) are not freed. No cleanup hook; depends on process exit to reclaim heap.
