# code/botlib/be_aas_optimize.c — Enhanced Analysis

## Architectural Role

This file implements a **post-load, one-time geometry compaction pass** in the AAS initialization pipeline. It runs after `AAS_LoadAASFile` completes and is specific to the **runtime botlib** (not the offline `bspc` compiler). Its sole purpose: strip all non-ladder-bearing geometry to reduce memory footprint, since only ladder faces are needed by the runtime reachability system after offline computation completes. This is a space-optimization pass that would be impossible during bspc compilation (which needs full BSP geometry for reachability calculation).

## Key Cross-References

### Incoming (who depends on this file)
- **`AAS_Setup` / `AAS_LoadFiles`** (`be_aas_main.c`) — calls `AAS_Optimize()` after loading BSP and AAS binary data, as the final step before the bot AI becomes operational
- **`botimport.Print`** (server-supplied via `botlib_import_t`) — receives status message on completion

### Outgoing (what this file depends on)
- **`aasworld`** global (`be_aas_def.h`) — reads all geometry arrays (`vertices`, `edges`, `faces`, `edgeindex`, `faceindex`, `areas`, `reachability`); rewrites them in-place via `AAS_OptimizeStore`
- **Memory subsystem** (`l_memory.h`) — `GetClearedMemory`, `FreeMemory` for transient `optimized_t` workspace and final deallocation
- **String utilities** (`q_shared.h`) — `Com_Memcpy` for bulk struct copying, `VectorCopy` for per-vertex duplication
- **Reachability type constants** (`be_aas_def.h`) — `TRAVEL_ELEVATOR`, `TRAVEL_JUMPPAD`, `TRAVEL_FUNCBOB`, `TRAVELTYPE_MASK` for special-case skipping

## Design Patterns & Rationale

### Lazy Deduplication via Index Remapping
The code uses three parallel lookup tables (`vertexoptimizeindex[]`, `edgeoptimizeindex[]`, `faceoptimizeindex[]`) to avoid duplicating shared geometry. When multiple ladder faces reference the same edge, `AAS_OptimizeEdge` checks if that edge was already copied, reusing its new index instead. This is a classic **compaction pattern** dating to pre-1990s C; modern engines would likely use a hash map of (old_id → new_id) pairs.

### Preservation of Sign Encoding
The code is meticulous about preserving sign bits throughout (`if (edgenum > 0) return optedgenum; else return -optedgenum;`). This reflects a **compression optimization**: direction/orientation is packed into the sign rather than stored as a separate flag. Since this is a single-frame optimization pass, not a hot-path operation, this level of micro-optimization seems inherited from offline tooling rather than runtime necessity.

### Dummy Element Reservation
Both `numedges` and `numfaces` are initialized to 1, reserving index 0 as a sentinel. This prevents off-by-one bugs and allows distinguishing "not found" (0) from "found at first position" (1). It's a **convention borrowed from compiler symbol tables** and is idiomatic to Quake-era code.

### Hierarchical Recursion (Area → Face → Edge → Vertex)
The optimization walks the hierarchy top-down: `AAS_Optimize` → `AAS_OptimizeArea` → `AAS_OptimizeFace` → `AAS_OptimizeEdge`. Each level filters and remaps, building compacted arrays bottom-up. This mirrors the original BSP→AAS construction in `bspc` and ensures referential integrity.

### Hardcoded Game-Specific Logic
The reachability index fixup skips three travel types because they encode **semantic data, not geometry**. This violates modularity (the optimizer now depends on game rules) but saves memory by not tracking travel-type-specific edge/face pointers. It's a pragmatic tradeoff.

## Data Flow Through This File

**Inputs** (from `aasworld` before optimization):
- Full BSP geometry: potentially thousands of vertices, edges, faces (most non-ladder)
- Reachability records linking travel types to face/edge indices

**Transformation**:
1. Allocate parallel compacted arrays (upper-bounded by original sizes)
2. Iterate all areas; for each, iterate all faces; filter by `AAS_KeepFace` (ladder-only)
3. Recursively copy kept faces and their edges; recursively copy referenced vertices
4. Build new index → old index maps for later patching
5. Patch all reachability records, skipping special travel types

**Outputs** (to `aasworld` after optimization):
- Compacted geometry: typically 10–20% of original size (rough estimate; only ladder faces + their incident edges/vertices remain)
- Reachability records with remapped indices
- Old geometry arrays freed

**Key state transition**: `aasworld.vertices` → `aasworld.numvertexes` (possibly halved or smaller) in one atomic swap.

## Learning Notes

### What a Developer Studies Here
1. **Compaction patterns**: How to efficiently deduplicate and reindex data structures using parallel lookup tables
2. **Sign-bit encoding**: A memory optimization technique where direction/sign is packed into a pointer or integer, common in pre-2000s engines but rarely used today
3. **Hierarchical filtering**: How to traverse nested data structures (area → face → edge → vertex) and selectively prune
4. **One-time initialization passes**: When and how to do expensive data massaging at load time rather than per-frame
5. **Sentinel/dummy elements**: Using index 0 as "not found" or "invalid" to avoid null-pointer checks

### Idiomatic to This Engine/Era
- **No dynamic allocation tracking**: No RAII, no smart pointers; manual free/malloc with a few lines of copying
- **Linear arrays as primary data structure**: No hash tables, no trees (except BSP which is implicit in the offline phase); everything is integer-indexed
- **Batch struct copying with `Com_Memcpy`**: Efficient because structs are fixed-size and tightly packed
- **Callback/predicate pattern** (`AAS_KeepFace`, `AAS_KeepEdge`): Allows filtering logic to be swapped without refactoring; though here the predicates are trivial

### Modern Parallels
- **Modern engines use ECS**: Entity data is shredded into SOA (struct-of-arrays) rather than AOS (array-of-structs), enabling SIMD-friendly iteration. This code is still AOS.
- **Scene graphs vs flat arrays**: Quake III keeps everything in flat index-based arrays for cache locality; modern engines often use tree structures for spatial hierarchy.
- **Lazy evaluation**: Modern compilers (LLVM) and JITs often defer optimizations to load-time or runtime; this pass is intentionally eager to amortize cost.

## Potential Issues

### 1. Hardcoded `FACE_LADDER` Filter
The function `AAS_KeepFace` returns 1 only if `face->faceflags & FACE_LADDER`. If the offline compiler (`bspc`) ever marked ladder faces incorrectly, the optimizer would silently discard valid geometry, causing reachability errors. There's no logging or assertion to catch this. **Risk: moderate; mitigation: offline tooling testing.**

### 2. Reachability Index Patching Fragility
Three travel types (`TRAVEL_ELEVATOR`, `TRAVEL_JUMPPAD`, `TRAVEL_FUNCBOB`) skip index remapping. If future code ever interprets `facenum` or `edgenum` for these types as a geometry index (rather than semantic payload), silent data corruption would result. **Risk: low if travel types are stable; moderate if they're extended.**

### 3. No Validation of Compacted Data
After `AAS_OptimizeStore` swaps pointers, there's no verification that the compacted world is geometrically consistent (e.g., no dangling edge references, valid vertex indices). A stray write during compaction could corrupt `aasworld.areas` and cause crashes later. **Risk: low if the compaction logic is correct; moderate if extended.**

### 4. Quadratic Behavior in Edge Deduplication
For each face, each edge is checked against `edgeoptimizeindex[]` linearly. With N faces and M edges per face, the complexity is O(N×M). For typical maps, this is negligible (optimization pass runs once at load), but very large maps or faces with thousands of edges could see slowdown. **Risk: negligible for current use case.**

---

## Summary

This file is a **focused, well-isolated post-load compaction pass** that trades runtime memory for one-time CPU cost at AAS initialization. Its design reflects early 2000s constraints (memory was precious, CPU cycles abundant at load time) and Quake III's preference for flat index-based data structures over pointers. It's a good example of how to preserve sign-bit encoding and deduplicate hierarchical data, but its hardcoded game-specific logic (travel types, ladder flag) makes it less flexible than a general-purpose optimizer. Modern engines would likely profile to decide whether this optimization is worth the complexity, but in Quake III's era, it was a sensible way to minimize the on-disk AAS binary size.
