# code/bspc/aas_facemerging.c — Enhanced Analysis

## Architectural Role
This file implements two consecutive geometry optimization passes in the **offline BSP→AAS compiler pipeline** (running within `bspc`). It reduces face count in the temporary `tmpaasworld` after initial area creation but before reach computation and clustering, enabling the final AAS world to be more compact. These functions are internal to offline compilation and have no runtime equivalent—the final AAS file (`aasfile.h` format) discards the winding geometry entirely, storing only connectivity and metadata.

## Key Cross-References

### Incoming (who depends on this file)
- **`aas_create.c`** (the main BSP→AAS converter pipeline) calls `AAS_MergeAreaFaces` and `AAS_MergeAreaPlaneFaces` as consecutive optimization passes after `AAS_Create` populates the temporary world
- No other subsystem imports from this file; it is exclusive to the bspc offline tool

### Outgoing (what this file depends on)
- **`aas_create.h`**: imports `tmp_face_t`, `tmp_area_t`, `tmpaasworld` global, and mutation helpers (`AAS_RemoveFaceFromArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`)
- **`l_poly.c`** (via qbsp.h): calls `MergeWindings` and `TryMergeWinding` — the core winding merge algorithms that determine whether two face geometries can be fused
- **`map.c`** (via qbsp.h): provides `mapplanes[]` array for plane normals
- **`Log_Write`, `qprintf`**: logging/progress feedback to the compiler console

## Design Patterns & Rationale

### Two-Strategy Merge Approach
The file implements **two distinct merging strategies** exposed as separate passes:

1. **General pairwise merge** (`AAS_MergeAreaFaces`): For each area, tries all face pairs in a restart-on-success loop. This is **conservative but thorough**—it retries the entire area after each successful merge (using `lasttmparea` to backtrack), ensuring no mergeable pair is skipped due to order dependencies. Trade-off: potentially O(n²) in pathological cases, but conceptually simple and correct for offline use.

2. **Aggressive per-plane merge** (`AAS_MergeAreaPlaneFaces`): Pre-filters with `AAS_CanMergePlaneFaces` to identify planes whose faces all share identical front/back areas and flags, then unconditionally merges them all at once. **Single-pass efficiency** at the cost of stricter preconditions. Used as a follow-up pass to mop up remaining same-plane faces.

**Rationale**: The first pass handles cases where not all coplanar faces are mergeable (e.g., some have solid on one side, others don't). The second pass handles the easy wins—entire groups of compatible coplanar faces that `AAS_CanMergePlaneFaces` pre-validates.

### Winding Merge Polymorphism
`AAS_TryMergeFaces` branches on whether both areas are "real" (non-zero):
- **Both areas exist**: calls `MergeWindings` (unconstrained convex hull merge, potentially expensive but correct when both sides are free space)
- **Solid on one side**: calls `TryMergeWinding` (strict edge-sharing only, fails if the faces can't merge cleanly along a shared edge)

**Rationale**: Winding merge is fragile. When one side is solid (`backarea == 0`), the winding must remain valid for pathfinding — a full hull merge could produce incorrect geometry. Strict edge-sharing is safer.

### Iterator Invalidation Handling
In `AAS_MergePlaneFaces`, the loop saves `nextface2` **before** processing the merge, because `AAS_FreeTmpFace(face2)` invalidates `face2->next`. This is defensive coding typical of offline tools where correctness matters more than elegance.

## Data Flow Through This File

```
Input: tmpaasworld
  ├─ areas (linked list)
  └─ Each area's tmpfaces (doubly-linked list of faces)

Step 1: AAS_MergeAreaFaces()
  ├─ For each area:
  │  └─ Try all face pairs (face1, face2)
  │     ├─ Call AAS_TryMergeFaces(face1, face2)
  │     │  ├─ Check flags, areas, plane match
  │     │  ├─ Call MergeWindings or TryMergeWinding
  │     │  ├─ Replace face1->winding with merged result
  │     │  ├─ Remove face2 from its areas (AAS_RemoveFaceFromArea)
  │     │  ├─ Free face2 (AAS_FreeTmpFace)
  │     │  └─ Return true/false
  │     └─ If successful, restart area (backtrack to lasttmparea)
  └─ Output: Some faces merged, face count reduced

Step 2: AAS_MergeAreaPlaneFaces()
  ├─ For each area:
  │  └─ For each plane:
  │     ├─ Call AAS_CanMergePlaneFaces (read-only check)
  │     │  └─ Verify all coplanar faces have matching front/back/flags
  │     └─ If all compatible:
  │        └─ Call AAS_MergePlaneFaces (aggressive merge)
  │           ├─ Merge all coplanar faces into the first one
  │           ├─ Free all others
  │           └─ Update area face list
  └─ Output: Coplanar faces grouped; final face count lower

Final: Modified tmpaasworld is passed to next stage (reach computation)
```

## Learning Notes

### Offline vs. Runtime Mindset
This file exemplifies **offline tool pragmatism**:
- Heavy mutation and freeing of temporary structures (no GC, no lifetime tracking)
- Restart-on-success loops (simple correctness over batch efficiency)
- Debug assertions (`#ifdef DEBUG`)
- Progress logging via `qprintf` and `Log_Write`

Modern game engines would handle this differently (e.g., immutable passes, streaming stages, memory pools), but for a 2005 offline compiler this is idiomatic.

### Winding Merge Fragility
The distinction between `MergeWindings` and `TryMergeWinding`, and the commented-out "alternate implementation," reveals that **convex polygon merging is subtle**. The code doesn't just blindly merge all coplanar faces; it checks preconditions and falls back to stricter algorithms when needed. This is a key lesson: geometry operations that seem simple (merge two polygons) are actually difficult under constraints.

### Pipeline Architecture
The broader BSP→AAS pipeline (visible in the cross-reference list) is **multi-stage**:
1. **Area creation** (BSP leaf → convex regions)
2. **Face merging** (reduce redundant faces) ← **This file**
3. **Area merging** (merge adjacent areas)
4. **Edge melting** (align adjacent face edges)
5. **Reach computation** (inter-area connectivity)
6. **Clustering** (spatial partitioning for fast queries)
7. **Optimization** (strip unused geometry, compress)

Each pass is a separate file/module, applied sequentially. This **staged approach** allows each optimization to be tested and tuned independently—compare with modern ECS/data-driven engines, which often interleave such operations.

### Determinism
Face merging happens during offline compilation, so non-determinism isn't a concern. The restart-on-success strategy, while slower, guarantees finding all mergeable pairs regardless of link-list traversal order—important for reproducible AAS generation.

## Potential Issues

1. **Commented-out code (lines 94–137)**: A large alternate `AAS_TryMergeFaces` implementation is present but disabled. Its condition `(face1->planenum & ~1) != (face2->planenum & ~1)` suggests it merged opposite-facing planes (flipped faces). The comment `"face %d and %d, same front and back area but flipped planes"` hints this was intentionally removed. **Recommendation**: Document why flipped-plane merging was disabled, or clean up the dead code if it's no longer relevant.

2. **Restart-on-success scaling**: In `AAS_MergeAreaFaces`, the restart loop could be pathological for dense or poorly-structured geometry. For typical Q3 maps this is fine, but maps with many mergeable faces per area might see significant slowdown. No explicit complexity guarantee is documented.

3. **Implicit assumption on `mapplanes[]`**: Both `AAS_TryMergeFaces` and `AAS_MergePlaneFaces` index `mapplanes[face->planenum]` without bounds checking. If `face->planenum` is ever invalid or out of bounds, a crash results. This assumes upstream code (`AAS_Create`) always assigns valid plane indices.

4. **Silent winding merge failure**: `TryMergeWinding` can fail (return null) when strict edge-sharing is impossible. The code then returns `false`, but logs nothing. For debugging a map that doesn't merge as expected, this silent failure could be confusing. Consider adding debug-level logging when `TryMergeWinding` fails.
