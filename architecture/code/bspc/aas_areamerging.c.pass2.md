# code/bspc/aas_areamerging.c — Enhanced Analysis

## Architectural Role

This file implements a **post-initial-creation optimization pass** in the offline BSPC AAS compilation pipeline. It sits between the raw temporary area generation (`aas_create.c`) and final storage (`aas_store.c`), reducing area fragmentation by merging adjacent areas that satisfy convexity and content constraints. As a build-time tool within BSPC (not runtime), it never executes during gameplay; instead, it shrinks the pre-compiled AAS file footprint and improves bot pathfinding efficiency by collapsing microfragments into navigable units.

## Key Cross-References

### Incoming (who depends on this file)
- **`aas_create.c:AAS_Create`** — The main AAS initialization pipeline invokes `AAS_MergeAreas()` after initial area generation and before clustering. The merged result flows into subsequent passes.
- **`aas_create.h`** — Provides function declarations for this module and headers for dependent utilities.

### Outgoing (what this file depends on)
- **`aas_create.c`** — Calls `AAS_AllocTmpArea`, `AAS_RemoveFaceFromArea`, `AAS_AddFaceSideToArea`, `AAS_FreeTmpFace`, `AAS_CheckArea`, `AAS_FlipAreaFaces`, `AAS_GapFace` (all area/face management primitives).
- **`qbsp.h`** — Global `mapplanes` array, `plane_t` struct, `winding_t`, `DotProduct` macro, `Error()`, `qprintf()`.
- **`aasfile.h`** — Flag constants `FACE_GROUND`, `FACE_GAP` marking surface behavior.
- **Global `tmpaasworld`** (from `aas_create.c`) — The entire in-progress temporary AAS structure; iterated over to find mergeable area pairs.

## Design Patterns & Rationale

### Two-Phase Merge with Prioritization
The outer `while(1)` loop in `AAS_MergeAreas` alternates between two passes:
1. **Ground-first pass** (`groundfirst=true`) — Merges only areas that have at least one ground face (`FACE_GROUND`). This prioritizes structural coherence for gameplay (bots perceive ground areas as safer, more navigable).
2. **All-areas pass** (`groundfirst=false`) — Attempts merging any remaining valid pair.

This heuristic **reduces non-convex jitter** from aggressive merging of unrelated spaces early, while still achieving compaction.

### Forward-Pointer Redirection Chain
Instead of immediately splicing out old areas, the code marks them `invalid=true` and sets `mergedarea` pointers:
```c
tmparea1->mergedarea = newarea;
tmparea1->invalid = true;
```
The post-merge tree refresh (`AAS_RefreshMergedTree_r`) follows these chains with a `while` loop:
```c
while(tmparea->mergedarea) tmparea = tmparea->mergedarea;
```
This decouples merge logic from tree topology, allowing multiple merges to chain transitively (A→B→C) without per-merge tree surgery.

### Epsilon-Based Convexity Guard
The `NonConvex` function uses dot-product plane tests with a `CONVEX_EPSILON = 0.3` tolerance, checking whether any face vertex from one area lies *behind* the opposite area's plane. This is:
- **Simple**: Avoids heavyweight geometric queries (e.g., full winding intersection).
- **Conservative**: Rejects marginal merges to prevent subtle non-convex regions.
- **Pragmatic**: The 0.3-unit slack accommodates floating-point rounding without rejecting valid merges.

### Why This Structure?
The code is structured this way because:
1. **Iterative refinement** — Multiple passes allow compound simplifications (one merge may enable another).
2. **Offline safety** — No per-frame performance constraint; can afford O(n²) area-pair iteration.
3. **Gameplay semantics** — Prioritizing ground areas encodes domain knowledge about bot navigation.
4. **Lazy tree updates** — Deferring tree fixup to a single post-merge pass is faster than updating during each merge.

## Data Flow Through This File

```
tmpaasworld.areas (raw temporary areas from aas_create.c)
    ↓
[AAS_MergeAreas iterates all areas]
    ↓
[For each area's faces, test adjacent area pairs]
    ↓
[NonConvex + flag checks] ─→ Reject if unsafe
    ↓
[AAS_TryMergeFaceAreas: allocate newarea, migrate faces, mark old as invalid]
    ↓
[Both old areas: set .mergedarea = newarea, .invalid = true]
    ↓
[Loop continues until no merges occur in both passes]
    ↓
[AAS_RefreshMergedTree_r: walk BSP tree, follow .mergedarea chains in area leaves]
    ↓
tmpaasworld.areas (consolidated; old areas still in list but invalid)
```

**Key state transitions for each area:**
- `valid` → `(valid, contains old faces)` → `(invalid, mergedarea set)` → *removed from use by tree refresh*

## Learning Notes

### Engine-Specific Idioms
- **Temporary/Final distinction**: The BSPC pipeline uses `tmp_*` types for compile-time mutable structures, vs. final read-only structs in the compiled AAS file. This allows aggressive rewriting without per-operation serialization.
- **Chained allocation/linking**: Face and area management via `AAS_*SideToArea` / `AAS_RemoveFaceFromArea` reflects the era's pragmatic C approach (no containers, manual linked-list splicing).
- **Post-hoc tree refresh**: Separating geometry optimization from topology fixup is a common pattern in offline tools to avoid cascading updates.

### Contrast to Modern Engines
- **No ECS/components**: Areas and faces are monolithic C structs with embedded linked lists, not modular component types.
- **No spatial hashing/grid**: Merging relies on face adjacency iteration, not spatial query acceleration.
- **Build-time only**: Modern engines often compute AAS-like structures at load time with JIT caching; Q3A builds once and ships static `.aas` files.

### Geometric Insight
The `NonConvex` epsilon test teaches a lesson in **conservative convexity checking**: you don't need perfect topology (complex winding intersection); a simple plane-inequality check with slack catches most pathological merges and is fast enough for offline tools.

## Potential Issues

1. **Epsilon sensitivity**: `CONVEX_EPSILON = 0.3` is hardcoded. Maps with very tight geometry (narrow corridors, thin walls) might reject valid merges or accept invalid ones if the epsilon is poorly tuned. There's no diagnostics or logging to flag borderline cases.

2. **Chain traversal depth**: If merges form long chains (A→B→C→D→...), the `while` loop in `AAS_RefreshMergedTree_r` walks the entire chain per leaf. In pathological cases (sequential merging of 100s of areas), this could be O(n²) in total tree-fixup time. Mitigation: path compression (direct chain to final area) would be a one-line fix, but isn't implemented.

3. **Ground-first heuristic is opaque**: The hardcoded two-pass approach (ground then all) has no configuration knobs or explanation in code comments. If a map has unusual geometry (e.g., all-air level), the heuristic may be suboptimal. A configurable priority or user-selectable merge strategy would improve flexibility.

4. **No diagnostics**: There's commented-out `Log_Print` calls for debug output; if a merge silently fails due to ground/gap conflict or non-convexity, there's no trace. A `verbose` mode would aid debugging map-specific merge failures.
