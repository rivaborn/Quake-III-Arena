# q3map/qbsp.h — Enhanced Analysis

## Architectural Role

This header defines the **offline BSP map compiler**, an entirely separate tool from the runtime engine. It orchestrates the multi-phase compilation pipeline: parse brushes/patches → construct brush BSP tree → generate portals → build face BSP → allocate lightmaps → emit `.bsp` file. While the runtime engine (`code/qcommon/cm_load.c`) *loads* pre-compiled BSP data, this module defines the structures and orchestration for *generating* that data from source maps. The two-phase architecture (structural brush-BSP followed by visibility-aware face-BSP) distinguishes Q3's approach: portals drive visibility clustering rather than PVS-from-scratch.

## Key Cross-References

### Incoming (who depends on this file)
- **All `q3map/*.c` modules** include this header: `brush.c`, `map.c`, `csg.c`, `brushbsp.c`, `facebsp.c`, `portals.c`, `writebsp.c`, `surface.c`, `patch.c`, etc.
- The definitions here (type struct, extern globals, function declarations) form the contract for all q3map compilation stages
- No runtime engine code (`code/qcommon`, `code/server`, etc.) depends on this—the `.bsp` file format is the interface, not these structs

### Outgoing (what this file depends on)
- **Common utility headers** (`common/cmdlib.h`, `common/mathlib.h`, `common/polylib.h`, `common/bspfile.h`)
- **q3map-specific headers** (`shaders.h`, `mesh.h`)
- Implicitly drives the static link against `common/` offline tools library (no runtime engine link)

## Design Patterns & Rationale

**Two-Phase BSP Construction:**
1. **Brush BSP** (`node_t`/`bspbrush_t`): Hierarchical spatial subdivision using brush planes; separates structural/detail geometry.
2. **Face BSP** (`bspface_t`/`tree_t`): Visibility-aware re-tessellation of all faces; portals computed here drive PVS clustering.

**Rationale**: Brush-based phase is fast, canonical, and matches editor representation. Face-based phase adds visibility grouping—portals naturally emerge from coplanar face boundaries. This two-step avoids expensive portal computation from raw brushes.

**Portal-Centric Visibility**: Portal generation (`MakeHeadnodePortals`, `SplitNodePortals`, `FloodEntities`) is the crux. Portals separate leaf clusters and determine which faces can "see" adjacent areas. At runtime, the compiled PVS and area-portal data in the `.bsp` file (written by `writebsp.c`) enable fast frustum culling and audibility.

**Compile-Time Lightmap Allocation**: Unlike modern engines, lightmaps are allocated offline (`AllocateLightmaps`). The `mapDrawSurface_t` struct encodes lightmap UVs, dimensions, and origin—all pre-computed. Runtime surfaces reference these static atlases.

**Brush Primitive Versioning**: Constants `BPRIMIT_OLDBRUSHES` vs `BPRIMIT_NEWBRUSHES` and the `texMat[2][3]` / `vecs[2][4]` dual representation in `side_t` suggest migration from Q2-style texture coordinates to Q3's matrix-based primitive mode. This is a backward-compat shim.

## Data Flow Through This File

1. **Input**: Map file (entity strings, brush/patch data)
2. **Parsing**: `LoadMapFile` → `FinishBrush` populates `buildBrush`; patches become `parseMesh_t`
3. **Structural Phase**: `BrushBSP(brushlist)` builds `node_t` tree; brushes recursively split by planes
4. **Portal Phase**: `MakeTreePortals` → recursively creates `portal_t` linking leaf nodes → `FloodAreas` floods connectivity
5. **Face Phase**: `FaceBSP` builds separate visibility tree from structural face list
6. **Surface Prep**: `MakeDrawSurfaces` → `ClipSidesIntoTree` → `MergeSides` → `SubdivideDrawSurfs` refines per-shader surfaces
7. **Lightmaps**: `AllocateLightmaps` assigns UV space to each `mapDrawSurface_t`
8. **Output**: `BeginBSPFile` → `EndModel` (per entity) → `EndBSPFile` writes `dnode`, `dleaf`, `dface`, `dshader`, `dlightgrid` lumps

**Key state transitions**:
- `plane_t` arrays grow in `map.c::FindFloatPlane` → reused by `brushbsp.c::SplitBrush` 
- `node_t` tree built → portals created → faces collected → surfaces subdivided → output

## Learning Notes

**Q3 idioms absent from modern engines**:
- **Brush-based level design** (vs. pure mesh or voxel): Intuitive for architects; requires careful CSG handling to avoid artefacts.
- **Two-phase BSP** is *relatively* lightweight compared to PVS-from-scratch algorithms; reflects 2005 era when both compile time and runtime memory were precious.
- **Compile-time lightmap atlasing** (vs. dynamic/deferred): Guarantees repeatability; limits runtime flexibility (no real-time relighting).
- **Hint brushes** (`HINT_PRIORITY` and `hint` flags) are manual hints to reduce BSP depth—a micro-optimization rarely seen in modern pipelines.

**Architectural insights**:
- Portals are *not* pre-computed PVS; they're *intermediate* geometric entities from which PVS is derived (done elsewhere, possibly in `code/botlib` or a post-processing pass).
- The `opaque`, `backSide`, `bevel` flags in `side_t` encode semantic information that guides split plane selection—not purely geometric.
- Draw surfaces are *not* leaves of the BSP; they're independent lists (per-shader, per-entity), *clipped* into the tree for visibility culling. This indirection allows flexible surface organization independent of spatial hierarchy.

**Connection to runtime**:
- Compiled `.bsp` format (not these structs) is the contract. `code/qcommon/cm_load.c` deserializes it into runtime `cnode_t`, `cleaf_t`, etc.—structurally similar but *not* identical.
- PVS/PHS (used by renderer via `CM_ClusterPVS`) is written by `writebsp.c` during compilation; offline algorithms produce it, not runtime code.

## Potential Issues

- **Hard-coded limits**: `MAX_BUILD_SIDES 300`, `MAX_BRUSH_SIDES 1024`, `MAX_PATCH_SIZE 32` are static—no graceful overflow handling visible. Maps exceeding these would silently truncate or crash.
- **Global state**: Many externs (`mapplanes`, `buildBrush`, `mapent`) are mutable globals accessed from multiple modules—difficult to parallelize or refactor.
- **Brush primitive ambiguity**: Dual texture representation (`texMat` and `vecs`) in `side_t` suggests incomplete migration; unclear when each is valid.
- **Draw surface interdependency**: The `FilterDrawsurfsIntoTree` and `ClipSidesIntoTree` dual filtering suggests potential for inconsistency if one is out of sync with the other.
