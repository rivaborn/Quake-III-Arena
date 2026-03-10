# code/bspc/map_hl.c

## File Purpose
Converts Half-Life BSP files into the BSPC tool's internal map brush representation. It reconstructs solid geometry by recursively splitting a world-bounding brush along BSP node planes, then applies texture information and merges adjacent brushes before emitting final `mapbrush_t` entries.

## Core Responsibilities
- Load and parse a Half-Life BSP file and its entities
- Classify texture names to content types (solid, water, lava, slime)
- Recursively reconstruct `bspbrush_t` geometry from the HL BSP tree
- Assign texture/texinfo data to brush sides by matching face overlap
- Split brushes at face boundaries when multiple textures conflict on a side
- Merge compatible adjacent brushes to reduce brush count
- Convert `bspbrush_t` records into `mapbrush_t` entries, optionally invoking AAS brush creation

## Key Types / Data Structures
None defined locally; uses types from `qbsp.h` and `l_bsp_hl.h`.

| Name | Kind | Purpose |
|---|---|---|
| `bspbrush_t` | struct (external) | Intermediate brush with windings per side, used during reconstruction |
| `mapbrush_t` | struct (external) | Final map-format brush stored in global `mapbrushes[]` array |
| `hl_dleaf_t` | struct (external) | HL BSP leaf with contents classification |
| `hl_dnode_t` | struct (external) | HL BSP node with plane number and child indices |
| `hl_dface_t` | struct (external) | HL BSP face used for texinfo assignment |
| `hl_dmiptexlump_t` / `hl_miptex_t` | struct (external) | HL texture lump and mip texture record |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `hl_numbrushes` | `int` | global | Running count of brushes created (debug/progress) |
| `hl_numclipbrushes` | `int` | global | Count of clip/untextured brushes discarded during conversion |

## Key Functions / Methods

### HL_TextureContents
- **Signature:** `int HL_TextureContents(char *name)`
- **Purpose:** Maps a HL texture name to a Q2-style content constant.
- **Inputs:** Texture name string.
- **Outputs/Return:** One of `CONTENTS_SOLID`, `CONTENTS_LAVA`, `CONTENTS_SLIME`, `CONTENTS_WATER`.
- **Side effects:** None.
- **Calls:** `Q_strncasecmp`
- **Notes:** Current contents (`!cur_*`), origin, clip, and translucent cases are commented out; any `!`-prefixed name not matched earlier defaults to `CONTENTS_WATER`.

### HL_SplitBrush
- **Signature:** `void HL_SplitBrush(bspbrush_t *brush, int planenum, int nodenum, bspbrush_t **front, bspbrush_t **back)`
- **Purpose:** Splits a brush along a map plane, producing front and back halves. Modified from the standard splitter to tolerate HL's many tiny leaves.
- **Inputs:** Source brush, plane index, BSP node index (stored in split side's `surf` for later texinfo lookup).
- **Outputs/Return:** Sets `*front` and `*back`; either may be NULL on degenerate splits.
- **Side effects:** Allocates new `bspbrush_t` via `AllocBrush`/`CopyBrush`; frees tiny-volume results; logs warnings via `Log_Print`.
- **Calls:** `CopyBrush`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingIsTiny`, `WindingIsHuge`, `BrushMostlyOnSide`, `AllocBrush`, `ClipWindingEpsilon`, `BoundBrush`, `FreeBrush`, `BrushVolume`, `CopyWinding`, `Log_Print`

### HL_SolidTree_r
- **Signature:** `int HL_SolidTree_r(int nodenum)`
- **Purpose:** Recursively tests whether an entire BSP subtree contains only solid/sky leaves.
- **Inputs:** BSP node index (negative = leaf).
- **Outputs/Return:** `true` if subtree is entirely solid; `false` otherwise.
- **Side effects:** None.
- **Calls:** Recursive self-calls; reads `hl_dleafs`, `hl_dnodes`.

### HL_CreateBrushes_r
- **Signature:** `bspbrush_t *HL_CreateBrushes_r(bspbrush_t *brush, int nodenum)`
- **Purpose:** Recursively descends the HL BSP tree, splitting the given brush at each node plane and assigning content types at leaves.
- **Inputs:** Brush covering current subtree volume; BSP node index.
- **Outputs/Return:** Linked list of `bspbrush_t` with contents set.
- **Side effects:** Frees input brush; allocates front/back brushes; calls `Error` on unknown contents.
- **Calls:** `HL_SplitBrush`, `FreeBrush`, `FindFloatPlane`, `Log_Print`, `Error`

### HL_CreateBrushesFromBSP
- **Signature:** `bspbrush_t *HL_CreateBrushesFromBSP(int modelnum)`
- **Purpose:** Entry point for BSP-to-brush reconstruction for a single model; creates the initial world-bounding brush and starts the recursive split.
- **Inputs:** Model index into `hl_dmodels`.
- **Outputs/Return:** Linked list of content-typed `bspbrush_t`.
- **Side effects:** Updates `map_mins`/`map_maxs` via `AddPointToBounds`; logs progress.
- **Calls:** `BrushFromBounds`, `VectorCopy`, `AddPointToBounds`, `HL_CreateBrushes_r`, `Log_Print`, `qprintf`

### HL_MergeBrushes
- **Signature:** `bspbrush_t *HL_MergeBrushes(bspbrush_t *brushlist, int modelnum)`
- **Purpose:** Iteratively merges adjacent brushes of identical content type until no further merges are possible.
- **Inputs:** Brush list; model number (0 enables progress printing).
- **Outputs/Return:** Reduced brush list.
- **Side effects:** Frees merged brushes; reallocates list structure.
- **Calls:** `TryMergeBrushes`, `FreeBrush`, `qprintf`

### HL_FaceOnWinding
- **Signature:** `float HL_FaceOnWinding(hl_dface_t *face, winding_t *winding)`
- **Purpose:** Computes the overlap area between a BSP face polygon and a brush-side winding.
- **Inputs:** HL face, brush-side winding.
- **Outputs/Return:** Overlap area (float); 0 if no overlap.
- **Side effects:** Allocates/frees temporary winding.
- **Calls:** `CopyWinding`, `ChopWindingInPlace`, `WindingArea`, `FreeWinding`

### HL_SplitBrushWithFace
- **Signature:** `bspbrush_t *HL_SplitBrushWithFace(bspbrush_t *brush, hl_dface_t *face)`
- **Purpose:** Splits a brush along each edge plane of a face to isolate the region covered by that face.
- **Inputs:** Brush to split; face defining the clip planes.
- **Outputs/Return:** List of new brushes; NULL on failure.
- **Side effects:** Frees on failure via `FreeBrushList`; logs via `Log_Print`.
- **Calls:** `SplitBrush`, `FindFloatPlane`, `FreeBrushList`, `FreeBrush`, `Log_Print`

### HL_TextureBrushes
- **Signature:** `bspbrush_t *HL_TextureBrushes(bspbrush_t *brushlist, int modelnum)`
- **Purpose:** Assigns `texinfo` to every untextured brush side by finding the BSP face with the greatest overlap; optionally splits brushes when conflicting textures exist on the same side.
- **Inputs:** Brush list; model number.
- **Outputs/Return:** Possibly restructured brush list with all sides textured.
- **Side effects:** Writes into `map_texinfo[]`; may split and free brushes; logs progress.
- **Calls:** `HL_FaceOnWinding`, `HL_SplitBrushWithFace`, `FreeBrush`, `qprintf`, `Log_Write`

### HL_FixContentsTextures
- **Signature:** `void HL_FixContentsTextures(bspbrush_t *brushlist)`
- **Purpose:** Ensures water/slime/lava brushes have a matching contents texture on all their sides; searches global `map_texinfo` for a matching texture if none of the existing sides qualify.
- **Inputs:** Brush list.
- **Side effects:** Modifies `texinfo` on brush sides; logs mismatches.
- **Calls:** `HL_TextureContents`, `Log_Print`

### HL_BSPBrushToMapBrush
- **Signature:** `void HL_BSPBrushToMapBrush(bspbrush_t *bspbrush, entity_t *mapent)`
- **Purpose:** Converts a single `bspbrush_t` into a `mapbrush_t` entry in the global arrays, invoking AAS or winding/bevel generation as needed.
- **Inputs:** Source bsp brush; owning map entity.
- **Side effects:** Increments `nummapbrushes`, `nummapbrushsides`, `hl_numclipbrushes`; writes to `mapbrushes[]` and `brushsides[]`; calls `AAS_CreateMapBrushes` or `MakeBrushWindings`+`AddBrushBevels`.
- **Calls:** `AAS_CreateMapBrushes`, `MakeBrushWindings`, `AddBrushBevels`, `Error`

### HL_CreateMapBrushes
- **Signature:** `void HL_CreateMapBrushes(entity_t *mapent, int modelnum)`
- **Purpose:** Top-level pipeline for one model: create → texture → fix contents → optionally merge → convert to map brushes.
- **Calls:** `HL_CreateBrushesFromBSP`, `HL_TextureBrushes`, `HL_FixContentsTextures`, `HL_MergeBrushes`, `HL_BSPBrushToMapBrush`, `FreeBrush`, `qprintf`

### HL_LoadMapFromBSP
- **Signature:** `void HL_LoadMapFromBSP(char *filename, int offset, int length)`
- **Purpose:** Top-level entry point: loads the HL BSP file, parses entities, iterates all entities to create map brushes for world and brush models.
- **Side effects:** Sets `loadedmaptype = MAPTYPE_HALFLIFE`; clears bounds; populates global map arrays.
- **Calls:** `HL_LoadBSPFile`, `HL_ParseEntities`, `ClearBounds`, `HL_CreateMapBrushes`, `ValueForKey`, `Log_Print`, `qprintf`

## Control Flow Notes
`HL_LoadMapFromBSP` is the external entry point called during BSPC's map-loading phase. It drives the full conversion pipeline: BSP load → entity parse → per-model brush reconstruction → texturing → merging → final map brush emission. No per-frame involvement; this is purely a one-shot offline processing stage.

## External Dependencies
- **Includes:** `qbsp.h` (types, globals, brush/plane utilities), `l_bsp_hl.h` (HL BSP structures and loader), `aas_map.h` (`AAS_CreateMapBrushes`)
- **Defined elsewhere:** `hl_dleafs`, `hl_dnodes`, `hl_dplanes`, `hl_dmodels`, `hl_dfaces`, `hl_texinfo`, `hl_dtexdata`, `hl_dedges`, `hl_dvertexes`, `hl_dsurfedges`, `hl_numfaces`, `hl_texdatasize` (all from `l_bsp_hl`); `mapplanes`, `map_mins`, `map_maxs`, `map_texinfo`, `map_numtexinfo`, `nummapbrushes`, `nummapbrushsides`, `mapbrushes[]`, `brushsides[]`, `entities`, `num_entities` (all from `map.c`/`qbsp.h`); `lessbrushes`, `nobrushmerge`, `create_aas` (from `bspc.c`)
