# code/bspc/aas_edgemelting.c — Enhanced Analysis

## Architectural Role

This file implements a **geometric refinement pass** in the offline `bspc` BSP-to-AAS compiler pipeline. It runs after temporary areas and faces are created from the BSP, and before area merging and final storage. Its purpose is to improve topological consistency of convex area boundaries by ensuring adjacent faces within the same area share vertices at their geometric boundary edges—a necessity because BSP faces (infinitely thin planes) may not have exact vertex coincidence when partitioned into finite AAS areas.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/aas_create.c`** — `AAS_Create()` main pipeline function (implied by first-pass analysis; not shown in cross-ref excerpt but typical bspc architecture) calls `AAS_MeltAreaFaceWindings()` after `AAS_CreateAreas_r()` completes.
- **Entry via `bspc.c`** — The offline tool's main loop invokes the full AAS generation pipeline, which includes this melting pass.

### Outgoing (what this file depends on)
- **Geometry utilities** — `PointOnWinding`, `AddWindingPoint`, `FreeWinding` (defined in `code/bspc/l_poly.c` or similar offline-tool geometry library, not in runtime `code/qcommon`).
- **Global state** — `tmpaasworld` (extern, from `aas_create.c`), `mapplanes[]` (extern, from `code/bspc/map.c`).
- **Logging** — `Log_Write()` (from `code/bspc/l_log.c`), `qprintf()` (console output utility).
- **Error handling** — `Error()` macro (from `code/bspc/l_cmd.c`, used only in `#ifdef DEBUG`).

## Design Patterns & Rationale

**Purpose-Driven Geometric Refinement**: The melting pass is a **brute-force O(N²) all-pairs pairwise face refinement**. For each area:
- Iterate all face pairs (nested loop over `tmparea->tmpfaces`)
- Check if any vertex of face2 lies *on an edge* of face1's winding
- If so, insert that vertex into face1 (splitting the edge)
- Repeat: each insertion modifies face1, so subsequent pairs see the growing winding

**Order Dependency**: The inner-loop reassignment `face1->winding = neww` after each insertion means the algorithm is **order-dependent**. The order in which face pairs are processed affects the final result. This is marked as `// FIXME: this is buggy` in the original code—likely because:
1. Non-commutativity: `MeltFaceWinding(A, B)` followed by `MeltFaceWinding(B, A)` may not converge to the same winding
2. Potential for missed vertices if the list is modified during traversal (though the linked-list traversal via `next[side]` is safe)
3. No guarantee that all pairwise boundary vertices are eventually inserted

**Why This Pattern?** In an **offline compilation tool**, correctness-by-brute-force is acceptable:
- No real-time budget pressure
- Reproducibility is more important than elegance
- The tool runs once during map compilation, not per frame

**Memory Strategy**: Heap alloc/free per vertex insertion (`FreeWinding` + `AddWindingPoint`) is expensive but acceptable offline. In a runtime engine, such per-vertex allocation would be unacceptable.

## Data Flow Through This File

**Input:**
- Temporary AAS world (`tmpaasworld`) with areas, each containing a linked list of faces with pre-computed windings
- Each winding is a polygon in 3D space attached to a plane in `mapplanes[]`

**Processing:**
1. `AAS_MeltAreaFaceWindings()` iterates all areas
2. For each area, `AAS_MeltFaceWindingsOfArea()` runs an O(N²) pairwise pass
3. For each pair, `AAS_MeltFaceWinding()` checks if face2's vertices lie on face1's boundary edges
4. If a vertex lands on an edge, insert it (heap allocate new winding, free old)

**Output:**
- Modified face windings (in-place mutations to `face->winding` pointers)
- Progress logging: split count per area, then total across all areas
- Return value: count of winding splits (used for logging only)

**Side Effects:**
- Heap fragmentation from repeated alloc/free of windings
- Mutation of global `tmpaasworld` state during compilation

## Learning Notes

**Idiomatic to this era (2005 id Tech 3):**
1. **Offline tool philosophy**: Complex geometric operations are acceptable if they're offline-only. Modern engines would precompute or use GPU geometry instancing.
2. **Linked-list traversal with side tracking** (`side1 = face1->frontarea != tmparea`): A compact pattern to iterate both front/back face lists for an area. Modern engines use doubly-linked lists or parent pointers.
3. **Heap-based geometry** (alloc per vertex): Acceptable for offline; modern engines would use arena allocators or static buffers.
4. **O(N²) pairwise algorithms**: No optimization for spatial locality. A modern approach might use edge hashing or spatial partitioning to detect boundary candidates.

**Connections to game engine concepts:**
- **Topological consistency**: Similar to how modern engines ensure manifold meshes (shared vertices, consistent face normals).
- **Geometric refinement**: Analogous to subdivision surface algorithms, though this is a one-pass boundary alignment rather than true subdivision.
- **Offline preprocessing**: Reflects the id Tech 3 philosophy of "compute once, use many times"—precompiled AAS geometry, precalculated lighting, precomputed clusters/portals.

## Potential Issues

1. **Marked as buggy** (`// FIXME: this is buggy`) — The algorithm is non-commutative and order-dependent. No evidence this causes practical failures in Q3A maps, but it's theoretically incomplete.
2. **No convergence check** — The nested loops assume a single pass suffices. If face pairs are processed in the wrong order, some boundary vertices might not be inserted until a second pass.
3. **Expensive memory pattern** — O(N²) pairs × O(M) vertices per pair × heap alloc per insertion. For large areas with many faces, this could be slow even for an offline tool.
4. **No validation** — No post-pass check that adjacent faces now share boundary vertices or that windings remain valid (convex, non-degenerate).

---

**Summary**: This file is a pragmatic offline geometric cleanup step, reflecting 2005-era game tooling priorities (offline correctness > elegance, brute force > optimization). Its position in the `bspc` pipeline is critical for ensuring the final AAS geometry has consistent boundaries, but the algorithm itself is acknowledged as incomplete.
