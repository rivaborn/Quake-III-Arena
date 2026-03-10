# code/bspc/aas_areamerging.h — Enhanced Analysis

## Architectural Role

This header is a critical compilation pass interface within BSPC's offline AAS generation pipeline. It sits at a strategic point: after `AAS_CreateAreas` (which generates initial areas from the BSP tree) and before `AAS_Optimize` (which compacts the final geometry). By merging adjacent compatible areas, this pass reduces the total area count and simplifies the navigation mesh that botlib will load at runtime, trading pre-computed complexity for faster runtime performance.

## Key Cross-References

### Incoming (who calls this file's functions)
- **BSPC main compilation pipeline** (`code/bspc/be_aas_bspc.c`): calls `AAS_MergeAreas` as one discrete compilation pass in `AAS_CalcReachAndClusters`
- **Related offline helper functions**: `AAS_TryMergeFaceAreas`, `AAS_MergeAreaFaces`, `AAS_MergeAreaPlaneFaces`, `AAS_RefreshMergedTree_r` (all in `aas_areamerging.c`)

### Outgoing (what this file depends on)
- **Global AAS state structures** (`aasworld`): mutates area, face, edge, vertex arrays populated by prior passes
- **Sibling compilation passes**: operates on data shaped by `AAS_CreateAreas` and `AAS_Create`; feeds output to `AAS_GravitationalSubdivision`, `AAS_MeltAreaFaceWindings`, `AAS_Optimize`
- **Face merging subsystem** (`aas_facemerging.c`): the "merge faces" operations are closely related; both aim to reduce geometric redundancy
- **Utility functions** (`aas_store.c`, `aas_map.c`): geometric and structural helpers for traversing and modifying the AAS graph

## Design Patterns & Rationale

**Multi-Pass Offline Compilation**: BSPC is designed as a sequence of discrete, stateless compilation passes operating on a single global `aasworld` structure. Each pass reads the output of the previous one and leaves its results for the next. This design allows:
- Modular verification (test each pass in isolation)
- Deterministic offline output (no randomness; reproducible `.aas` files)
- Reuse of shared AAS infrastructure between the offline compiler (BSPC) and runtime library (botlib)

**Area Consolidation Strategy**: Rather than merging during initial area creation (which would complicate that already-complex phase), the AAS pipeline first creates a conservative over-segmented mesh, then simplifies it post-hoc. This follows the principle of "separate concerns": create → analyze → optimize.

## Data Flow Through This File

**Input Data:**
- Global `aasworld`: populated with areas, faces, and edges created by `AAS_CreateAreas`
- Each area has flags, bounds, faces, and links to adjacent areas

**Transformation:**
- Iterates over area pairs checking compatibility (same floor material, similar geometry, reachability constraints)
- Merges compatible adjacent areas into single larger areas
- Updates face/edge references to point to merged areas
- Refreshes the area tree structure via `AAS_RefreshMergedTree_r`

**Output:**
- Modified `aasworld` with fewer, larger areas
- Passed to next compilation pass (gravitational subdivision, face merging, optimization)
- Eventually serialized to `.aas` file by `AAS_WriteAASFile`

## Learning Notes

**Quake III AAS Philosophy**: The AAS generation pipeline reveals a particular strategy for bot navigation: generate conservative initial areas, then aggressively merge safe areas to reduce runtime overhead. Modern engines (e.g., Unreal, Unity) use different approaches (hand-placed nav meshes, fully automated nav mesh generation with tuning parameters), but Quake III's offline-first design makes geometric analysis and optimization tractable.

**Code Reuse Pattern**: Notably, BSPC reuses botlib's AAS infrastructure (clusters, reachability, entity linking) via a stub adapter (`AAS_InitBotImport` in `be_aas_bspc.c`). This means the offline compiler and runtime library share implementations—a design choice that saves code but couples them tightly.

**Offline-Only Context**: Unlike botlib, BSPC's functions like `AAS_MergeAreas` don't exist in the runtime engine at all. There is no runtime merging; all topological simplification happens at compile time. This is an important architectural boundary.

## Potential Issues

None clearly inferable from this header alone; the implementation (`aas_areamerging.c`) would need review to assess correctness of merge heuristics, potential edge cases in graph traversal, or performance characteristics on large maps.
