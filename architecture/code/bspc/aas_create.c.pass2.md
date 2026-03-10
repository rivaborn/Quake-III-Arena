# code/bspc/aas_create.c — Enhanced Analysis

## Architectural Role
This file is the **BSP→AAS gateway** in the offline BSPC compiler—the initial conversion stage that transforms the compiled BSP tree into a temporary, face-classified intermediate representation. It orchestrates the full offline AAS pipeline (BSP construction → extraction → multi-pass refinement → serialization) and is never linked into the runtime engine. All subsequent AAS processing passes depend on the correctness of the temporary structures created here.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/bspc/bspc.c`** (`AASOuputFile`) — Calls `AAS_Create` as the sole entry point to AAS compilation after map loading
- **`aas_store.c`** — Consumes the completed `tmpaasworld` and serializes it to disk via `AAS_StoreFile`
- **`aas_gsubdiv.c`, `aas_facemerging.c`, `aas_areamerging.c`, `aas_edgemelting.c`, `aas_prunenodes.c`** — Each downstream pass modifies or replaces the temporary structures; pipeline orchestrated in `AAS_Create`

### Outgoing (what this file depends on)
- **`qbsp.h`** — Consumes `node_t`, `portal_t`, `plane_t`; BSP pipeline functions (`ProcessWorldBrushes`, `MakeTreePortals`, `FloodEntities`, `FillOutside`, `Tree_Free`); global `mapplanes[]` array
- **`aas_cfg.h`** — Reads `cfg.phys_gravitydirection`, `cfg.phys_maxsteepness` for face classification (ground vs. gap heuristics)
- **Geometry utilities** — `FreeWinding`, `ReverseWinding`, `CopyWinding`, `WindingCenter`, `WindingPlane`, `RemoveColinearPoints` from qbsp
- **Memory & logging** — `GetClearedMemory`, `FreeMemory`, `Log_Print`, `Log_Write`, `Error`, `qprintf`

## Design Patterns & Rationale

**Global Singleton + Doubly-Linked Lists:**  
`tmpaasworld` stores all in-progress state. Faces and areas are linked-list nodes with `prev[2]`/`next[2]` per-side traversal. This is idiomatic for offline tools where ownership is global and traversal order matters (the `side` parameter distinguishes which area owns the face, enabling efficient removal).

**Slab Allocation for Nodes:**  
`tmp_nodebuf_t` pools 128 nodes per slab to amortize allocations. This pattern (alloc-in-bulk, free-in-bulk) trades fragmentation for bulk deallocation via `AAS_FreeTmpAAS`.

**Classification-as-You-Extract:**  
Face flags (`FACE_GROUND`, `FACE_GAP`, `FACE_SOLID`, `FACE_LIQUID`, etc.) are assigned during `AAS_CreateArea`, not as a separate pass. This tightly couples BSP→AAS conversion to physics configuration, ensuring the temporary representation already reflects gameplay semantics.

**Multi-Pass Refinement:**  
Rather than create a perfect AAS tree in one pass, the design extracts a rough per-leaf decomposition, then iteratively merges, subdivides, and cleans up. This keeps individual passes simple.

## Data Flow Through This File

```
BSP Tree (node_t)
  ↓
AAS_CreateAreas_r(node)
  ├─ Recursively traverse tree
  ├─ Solid leaves → return NULL
  └─ Non-solid leaves → AAS_CreateArea(leaf)
       ├─ Allocate tmp_area_t
       ├─ For each portal → allocate tmp_face_t
       ├─ Classify faces (ground, gap, liquid, solid)
       ├─ Link faces to area via AAS_AddFaceSideToArea
       ├─ Validate winding orientation
       └─ Return tmp_node_t leaf
  ↓
tmpaasworld.areas (linked list of tmp_area_t)
tmpaasworld.faces (linked list of tmp_face_t)
tmpaasworld.nodes (tree of tmp_node_t)
  ↓
[Downstream passes: merging, subdivision, pruning]
  ↓
AAS_CreateAreaSettings() → aggregate face flags → tmp_areasettings_t
  ↓
AAS_StoreFile() → serialize to disk
```

## Learning Notes

**Offline-vs-Runtime Separation:**  
This is pure offline tooling—never linked into the engine. The `tmp_*` types are temporary, the global `tmpaasworld` is acceptable, and multi-pass refinement is fine because compilation happens once. Modern engines (e.g., Unreal, Unity) generate AAS-like data offline too.

**Physics-Driven Classification:**  
Face classification isn't just "read the BSP flag"; it's physics-aware. `AAS_GapFace` and `AAS_GroundFace` dot the face normal against inverse gravity to determine traversability. This couples the compiler to gameplay physics (`cfg`), reflecting the tight link between level design and bot AI pathfinding.

**Winding Orientation Nightmare:**  
The extensive winding validation code (`AAS_CheckFaceWindingPlane`, `AAS_FlipAreaFaces`, `AAS_CheckAreaWindingPlanes`) reveals a core pain point: ensuring every face's winding is consistent with its area's local frame (inside/outside). Modern engines often use signed distances or automatic handedness resolution; Q3's approach is manual and error-prone.

**Temporary ≠ Final Format:**  
Notice `tmp_face_t` and `tmp_area_t` are very different from the final `aas_file.h` structures. The conversion happens in `aas_store.c`, which iterates the tree and flattens it into arrays. This staging is intentional—temporary structures optimize for tree operations (parent/child pointers, linked lists), while final format optimizes for runtime speed (arrays, cache locality).

## Potential Issues

1. **No BSP Validation:** The pipeline trusts `ProcessWorldBrushes` output. A malformed BSP (e.g., non-convex leaves, degenerate portals) will silently create bad AAS.

2. **Winding Flipping Heuristic:** Face orientation validation uses area center projection, which can fail if the center is outside due to non-convex faces. `AAS_CheckFaceWindingPlane` tries to auto-correct but logs errors—relying on offline inspection.

3. **Global State Not Re-entrant:** `AAS_Create` can only run once per process; calling it twice without `AAS_FreeTmpAAS` leaks memory. The BSPC tool works around this by exiting after each map.

4. **Gravity Direction Dependency:** Face classification is hardcoded to use `cfg.phys_gravitydirection`. Maps compiled with non-standard gravity (e.g., side-gravity mods) may have misclassified geometry.
