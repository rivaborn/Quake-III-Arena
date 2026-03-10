# code/bspc/aas_facemerging.h — Enhanced Analysis

## Architectural Role
This header exposes the face-merging optimization layer within BSPC's offline AAS compilation pipeline. It sits midway in the geometry simplification sequence: after initial AAS areas are created from the BSP tree (via `aas_create.c`), these two functions consolidate redundant face geometry to reduce the footprint of the compiled `.aas` file before area-level merging (`aas_areamerging.h`) and final optimization (`be_aas_optimize.h`). The two-pass design (general area faces, then plane-constrained) reflects a staged approach to geometric reduction.

## Key Cross-References

### Incoming (who depends on this file)
- **BSPC pipeline drivers:** likely `code/bspc/be_aas_bspc.c` (`AAS_CalcReachAndClusters`) and/or `code/bspc/bspc.c` (main compilation orchestrator) call these functions as part of the post-BSP-load geometry optimization sequence
- **Related optimization modules:** `aas_edgemelting.h`, `aas_areamerging.h`, `be_aas_optimize.h` work in tandem to progressively reduce AAS geometry complexity

### Outgoing (what this file depends on)
- **Global AAS world state:** functions operate on structures populated by `aas_create.c` (temporary area/face/node storage) and `aas_store.c` (indexed AAS geometry store)
- **Geometric utilities:** implementation in `.c` likely uses face-winding logic, coplanarity tests, and polygon merge predicates (probably local helpers; cross-reference shows `AAS_CanMergePlaneFaces`, `AAS_MergePlaneFaces`, `AAS_TryMergeFaces` as internal functions)
- **Math & BSP primitives:** shared with `q_math.c`, plane representation, and boundary/area definitions from `be_aas_def.h`

## Design Patterns & Rationale
- **Two-pass optimization strategy:** `AAS_MergeAreaFaces` performs broad consolidation; `AAS_MergeAreaPlaneFaces` targets plane-coincident polygons for further reduction. This mirrors classical mesh-optimization pipelines (general passes followed by specific local passes).
- **Stateless public API:** both functions operate entirely on global AAS world state, requiring no parameters. This is idiomatic to BSPC's offline compilation model where the entire BSP+AAS context is held in module globals.
- **Geometric simplification before reachability:** face merging precedes reachability computation and alternative routing, ensuring bot-navigation structures operate on simplified geometry rather than redundant polygons.

## Data Flow Through This File
1. **Input:** AAS world populated by `AAS_Create` (via `aas_create.c`) containing redundant face geometry across all areas
2. **Processing:**  
   - `AAS_MergeAreaFaces`: scans all areas for mergeable faces (likely any coplanar or adjacent polygons)  
   - `AAS_MergeAreaPlaneFaces`: second pass consolidating only plane-coincident faces per area
3. **Output:** simplified AAS world with reduced face/edge counts, ready for area-merging and reachability computation
4. **Side effects:** modifies global area/face storage in place; counts and geometry indices shift as faces are merged

## Learning Notes
- **Bot-centric geometry optimization:** unlike a runtime renderer which loads pre-optimized assets, BSPC must optimize on-the-fly to keep AAS files manageable (bots need fast area queries and many-to-many reachability). Face merging is a key step in this offline budget.
- **Comparison to modern engines:** contemporary engines (e.g., Unreal, Unity) pre-optimize nav meshes in the editor or use hierarchical structures (quad-trees, navigation graphs); Q3A computes a flat AAS array and relies on geometric simplification + caching to achieve scalability.
- **Staged vs. monolithic:** separating area-level merging from plane-specific merging allows fine-grained control; a monolithic merge-all function would be harder to debug or tune independently.
- **Idiomatic BSPC pattern:** all offline AAS subsystems (create, merge, optimize, store) follow the same pattern: void functions operating on global `aasworld` singleton, populated during a sequential compilation phase.

## Potential Issues
- **No visibility into helper functions:** cross-reference shows `AAS_CanMergePlaneFaces`, `AAS_MergePlaneFaces`, `AAS_TryMergeFaces` are internal to `.c` file; cannot assess geometric correctness from header alone (e.g., are coplanarity tests robust? are windings preserved correctly after merge?).
- **Ordering dependency:** if `AAS_MergeAreaFaces` and `AAS_MergeAreaPlaneFaces` must be called in a specific sequence (which is likely), this constraint is implicit and undocumented in the public interface; a single `AAS_MergeFaces(...)` function with an explicit phase parameter might reduce integration risk.
- **State validation:** no pre/post-condition checks visible; callers must ensure AAS world is in a valid state before and after merging.
