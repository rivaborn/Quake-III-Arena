# code/bspc/qbsp.h

## File Purpose
Central shared header for the BSPC (BSP Compiler) tool used to build AAS (Area Awareness System) navigation data from map BSP files. It defines all core BSP data structures, build-time constants, global state declarations, and the full inter-module function API surface used across the BSPC pipeline.

## Core Responsibilities
- Defines fundamental BSP construction types (`plane_t`, `side_t`, `mapbrush_t`, `face_t`, `bspbrush_t`, `node_t`, `portal_t`, `tree_t`)
- Declares global build-control flags (`noprune`, `nodetail`, `nomerge`, `create_aas`, etc.)
- Declares global map storage arrays and counters (`mapplanes`, `mapbrushes`, `brushsides`, etc.)
- Enumerates supported map source formats (Q1, Q2, Q3, Half-Life, Sin)
- Provides the inter-module function declaration surface for: map loading, CSG, BSP construction, portalization, tree management, leak detection, GL debug output, and texture resolution
- Aggregates all BSPC-local headers into a single include point so translation units need only `#include "qbsp.h"`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `plane_t` | struct | Map plane: normal, distance, type, sign bits, hash chain for deduplication |
| `brush_texture_t` | struct | Texture mapping parameters (shift, rotate, scale, name, flags) for a brush side |
| `side_t` | struct | One face of a brush: plane index, texinfo, winding, content/surface flags |
| `mapbrush_t` | struct | Source brush from map file: entity/brush numbers, contents, AABB, sides array |
| `face_t` | struct | BSP output face with merge/split tracking, portal link, vertex indices |
| `bspbrush_t` | struct | Brush fragment during BSP construction: linked list node, AABB, side classification, variable-length sides |
| `node_t` | struct | BSP tree node or leaf: plane, children, portal list, brush list, cluster/area info |
| `portal_t` | struct | BSP portal connecting two nodes: plane, winding, side visibility, output face pair |
| `tree_t` | struct | Root of a BSP tree: head node, outside sentinel node, AABB |
| `map_texinfo_t` | struct | Extended texinfo for ME builds: S/T vectors, flags, texture name, animation chain |
| `textureref_t` | struct | Resolved texture reference: name, flags, value, contents, anim name |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `noprune`, `nodetail`, `fulldetail`, `nomerge`, `nosubdiv`, `nowater`, `noweld`, `noshare`, `notjunc`, `onlyents` | `qboolean` | global (extern) | Standard BSP build-step disable flags |
| `nocsg`, `create_aas`, `freetree`, `lessbrushes`, `nobrushmerge`, `cancelconversion`, `noliquids`, `capsule_collision` | `qboolean` | global (extern, ME) | BSPC-extended flags for AAS generation |
| `subdivide_size`, `microvolume` | `float`/`vec_t` | global (extern) | Geometry subdivision and culling thresholds |
| `outbase`, `source` | `char[]` | global (extern) | Output base path and source map path |
| `mapplanes[MAX_MAPFILE_PLANES]` | `plane_t[]` | global (extern) | All unique planes in the map |
| `nummapplanes`, `nummapbrushes`, `nummapbrushsides` | `int` | global (extern) | Counts for map geometry arrays |
| `mapbrushes[MAX_MAPFILE_BRUSHES]` | `mapbrush_t[]` | global (extern) | All source brushes |
| `brushsides[MAX_MAPFILE_BRUSHSIDES]` | `side_t[]` | global (extern) | All brush sides |
| `side_brushtextures[MAX_MAPFILE_BRUSHSIDES]` | `brush_texture_t[]` | global (extern) | Texture data per brush side |
| `map_texinfo[MAX_MAPFILE_TEXINFO]`, `map_numtexinfo` | `map_texinfo_t[]`/`int` | global (extern, ME) | Texture info table |
| `loadedmaptype` | `int` | global (extern) | Active map format (MAPTYPE_* constant) |
| `textureref[MAX_MAP_TEXTURES]` | `textureref_t[]` | global (extern) | Resolved texture table |
| `draw_mins`, `draw_maxs`, `drawflag` | `vec3_t`/`qboolean` | global (extern) | GL debug draw bounds and enable flag |
| `nodestack[NODESTACKSIZE]`, `nodestackptr`, `nodestacksize` | `int[]`/`int*`/`int` | global (extern, ME) | Stack for node traversal |

## Key Functions / Methods

All declarations are `extern` prototypes; implementations reside in other `.c` files.

### FindFloatPlane
- Signature: `int FindFloatPlane(vec3_t normal, vec_t dist)`
- Purpose: Returns an existing or newly-created plane index for the given normal/distance, deduplicating via hash.
- Inputs: Plane normal vector, plane distance.
- Outputs/Return: Integer index into `mapplanes[]`.
- Side effects: May add to `mapplanes[]` and increment `nummapplanes`.
- Calls: Defined in `map.c`.

### MakeBrushWindings
- Signature: `qboolean MakeBrushWindings(mapbrush_t *ob)`
- Purpose: Computes per-side convex windings for all sides of a source brush.
- Inputs: Pointer to a `mapbrush_t`.
- Outputs/Return: `qtrue` on success, `qfalse` if a side winding cannot be formed.
- Side effects: Allocates `winding_t` objects attached to each `side_t`.

### BrushBSP
- Signature: `tree_t *BrushBSP(bspbrush_t *brushlist, vec3_t mins, vec3_t maxs)`
- Purpose: Recursively partitions the brush list into a BSP tree.
- Inputs: Head of brush list, map AABB.
- Outputs/Return: Fully constructed `tree_t *`.
- Side effects: Allocates `node_t` and `bspbrush_t` fragments; major allocation phase.
- Calls: `SplitBrush`, `AllocNode`, `AllocBrush`, etc. (defined in `brushbsp.c`).

### MakeTreePortals
- Signature: `void MakeTreePortals(tree_t *tree)`
- Purpose: Generates portals for every node in the BSP tree to enable visibility/flood fill.
- Inputs: Completed BSP tree.
- Side effects: Allocates `portal_t` structures linked into `node_t::portals`.

### FloodEntities
- Signature: `qboolean FloodEntities(tree_t *tree)`
- Purpose: Flood-fills from entity positions to identify reachable leaf nodes and detect leaks.
- Outputs/Return: `qtrue` if all entities are inside sealed space.

### ProcessWorldBrushes
- Signature: `tree_t *ProcessWorldBrushes(int brush_start, int brush_end)`
- Purpose: High-level CSG + BSP pipeline entry point for the world brush model.
- Outputs/Return: Final `tree_t *` ready for AAS or BSP output.

### Tree_Free / Tree_Free_r
- Signature: `void Tree_Free(tree_t *tree)` / `void Tree_Free_r(node_t *node)`
- Purpose: Recursively release all nodes, portals, and brush lists in a tree.
- Side effects: Frees all associated heap allocations.

**Notes on trivial helpers:** `AllocNode`, `AllocBrush`, `CopyBrush`, `FreeBrush`, `FreeBrushList`, `BoundBrush`, `CountBrushList`, `BrushVolume`, `FreePortal`, `WindingIsHuge`, `WindingIsTiny` are allocation/query utilities documented by their signatures.

## Control Flow Notes

This header is the single include point for the BSPC compilation pipeline:
1. **Init**: `map.c` loaders (`Q1/Q2/Q3/HL/Sin_LoadMapFromBSP`) populate global `mapplanes[]`, `mapbrushes[]`, `brushsides[]`.
2. **CSG**: `csg.c` calls `MakeBspBrushList` → `ChopBrushes` → `ProcessWorldBrushes`.
3. **BSP**: `brushbsp.c` builds the `tree_t` via recursive `BrushBSP`.
4. **Portalization**: `portals.c` calls `MakeTreePortals` → `FloodEntities` → `FillOutside` → `FloodAreas` → `MarkVisibleSides`.
5. **Output / Leak**: `leakfile.c` and `glfile.c` consume the tree for diagnostics.
6. **Cleanup**: `tree.c` `Tree_Free` releases the tree; AAS generation (`create_aas`) is then driven from the result.

## External Dependencies

- `l_cmd.h` — command-line utilities, `qboolean`, file I/O helpers
- `l_math.h` — `vec3_t`, `vec_t`, vector math macros/functions
- `l_poly.h` — `winding_t`, convex polygon operations
- `l_threads.h` — threading and mutex primitives
- `../botlib/l_script.h` — lexical script parser (used by map loaders)
- `l_bsp_ent.h` — BSP entity (`entity_t`) type (defined elsewhere)
- `q2files.h` — Quake 2 BSP on-disk format definitions
- `l_mem.h`, `l_utils.h`, `l_log.h`, `l_qfiles.h` — memory, utility, logging, pak-file helpers
- `<io.h>` (Win32), `<malloc.h>` — platform allocation
- `quakefile_s` — forward-declared struct used by `LoadMapFromBSP` / `Q3_LoadMapFromBSP` (defined in `l_qfiles.h`)
- `tmp_face_s` — forward-declared in `portal_t`; defined in AAS/BSPC build code elsewhere
