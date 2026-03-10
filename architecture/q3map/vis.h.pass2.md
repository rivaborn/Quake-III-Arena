# q3map/vis.h — Enhanced Analysis

## Architectural Role

`vis.h` is the header for Q3A's visibility (PVS) pre-computation pipeline within the offline `q3map` BSP compiler. It defines the data structures and function declarations needed to compute potentially-visible-set (PVS) data by analyzing portal connectivity in the compiled BSP tree. The resulting PVS clusters and bit-encoded visibility tables are baked into the final `.bsp` file and consumed at runtime by the renderer (for frustum culling via `CM_ClusterPVS`) and server (for entity snapshot culling).

## Key Cross-References

### Incoming (callers)
- `q3map/bspc.c` main compilation pipeline invokes visibility computation after BSP tree construction
- Portal data structures populated by earlier `q3map` phases (brush → portal subdivision, possibly `vis.c` itself if this header is paired with implementation)

### Outgoing (dependencies)
- Lower-level types: `plane_t`, `winding_t` (shared with collision/BSP utilities)
- Global state: `portals[]`, `leafs[]` arrays populated by BSP phase; written to `vismap` buffer for serialization
- No direct engine dependencies; visibility is pure offline computation

## Design Patterns & Rationale

**Hierarchical Portal-based PVS**  
Portals subdivide BSP space into discrete visibility clusters. Rather than O(n²) cell-to-cell visibility, Q3A uses O(portal) transitive closure: if portal A can see portal B, and B can see C, then cluster A can access cluster C (indirectly).

**Flood-fill + Bit-string Optimization**  
`LeafFlow` and `*PortalVis` functions use depth-first traversal to propagate visibility through portal graphs. Visibility encoded as packed bit arrays (`portalvis[portals/8]`) trades computation time for storage efficiency—critical for pre-baking massive maps.

**Multi-stage Refinement Pipeline**  
`BasePortalVis` → `BetterPortalVis` → `PortalFlow` → `PassagePortalFlow` suggest progressively more accurate (and expensive) visibility passes, allowing trade-offs between compilation speed and runtime precision.

**Separator Plane Caching** (`#define SEPERATORCACHE`)  
Pre-computed separator planes cached in `pstack_t.seperators[][]` avoid redundant geometric tests during recursive portal traversal—hints at expensive plane-polygon intersection tests.

## Data Flow Through This File

**Input:** Portal graph structure (adjacency via `passage_t` chains; geometry via `winding_t`).

**Processing:**
- `LeafFlow(leafnum)`: Recursively flood-fill portals affecting a leaf.
- `BasePortalVis(portalnum)`: Compute which portals a given portal's leaf can "potentially see."
- `PortalFlow(portalnum)`: Refine via geometric visibility tests (winding clipping).
- `PassagePortalFlow`: Further refinement using multi-portal passages.
- `CreatePassages`: Build `passage_t` chain for efficient later queries.

**Output:** Bit-encoded visibility written to `vismap_p` buffer; serialized into BSP file's visibility lump.

**Runtime Path:** Renderer and server read this lump; `CM_ClusterPVS` queries it to determine which clusters are visible from a given cluster.

## Learning Notes

- **Portal-centric vs. Cell-centric:** Unlike Quake II (which used leaf-to-leaf visibility), Q3A uses portal-cluster hierarchies to reduce visibility data size while preserving accuracy.
- **Offline Pre-computation Trade-off:** Compilation cost is steep; visibility queries are near-instant. Modern engines prefer runtime frustum+occlusion culling; Q3A's static PVS was optimal for 1999 hardware.
- **Deterministic Recursion:** `pstack_t` stack frames maintain consistent winding geometry and depth context; no randomness ensures reproducible PVS data across builds.
- **Bit-packing Idiom:** `portalvis[portals/8]` is idiomatic for era; modern engines use GPU-friendly 2D or hierarchical structures.
- **Empirical Optimization:** `SEPERATORCACHE` and three `*PortalVis` variants suggest iterative refinement driven by real map performance profiling.

## Potential Issues

- **Fixed limits:** `MAX_PORTALS (32768)`, `MAX_PORTALS_ON_LEAF (128)`, `MAX_SEPERATORS (64)` can silently overflow or assert on unusually complex maps; no graceful degradation.
- **Recursion depth:** `pstack_t.depth` and recursive `*PortalVis` calls could exhaust stack on pathological portal graphs (highly interconnected, deep nesting).
- **No bounds on `passage_t` chains:** If a single portal leads to a leaf with many (>128) connected portals, passage chain iteration could become expensive.
