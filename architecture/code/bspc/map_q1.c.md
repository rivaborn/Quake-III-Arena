# code/bspc/map_q1.c

## File Purpose
Converts a Quake 1 (and Half-Life) BSP file into the BSPC tool's internal map brush representation. It reconstructs solid, water, slime, and lava brushes by recursively carving a bounding box with BSP node planes, then textures and optionally merges the result before forwarding to AAS or map export pipelines.

## Core Responsibilities
- Classify texture names to Q1 content types (solid, water, slime, lava)
- Recursively split a world-bounding brush along all BSP node planes to regenerate geometry (`Q1_CreateBrushes_r`)
- Assign texture info to brush sides by matching BSP faces that overlap each side's winding (`Q1_TextureBrushes`)
- Fix content-mismatched textures on liquid brushes (`Q1_FixContentsTextures`)
- Merge adjacent same-content brushes to reduce brush count (`Q1_MergeBrushes`)
- Convert internal `bspbrush_t` records into `mapbrush_t` entries for the global map arrays (`Q1_BSPBrushToMapBrush`)
- Orchestrate the full load pipeline per entity/model (`Q1_LoadMapFromBSP`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `bspbrush_t` | struct (typedef) | Internal brush with sides, windings, bounds; defined in `qbsp.h` |
| `mapbrush_t` | struct (typedef) | Exported map brush written into global `mapbrushes[]` array |
| `side_t` | struct (typedef) | One face of a brush: plane, texinfo, winding, flags |
| `q1_dleaf_t` | struct (typedef) | Q1 BSP leaf with content type |
| `q1_dnode_t` | struct (typedef) | Q1 BSP internal node with plane and child indices |
| `q1_dface_t` | struct (typedef) | Q1 BSP face with edge list, texinfo, planenum |
| `q1_miptex_t` | struct (typedef) | Q1 mip texture header containing the texture name |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `q1_numbrushes` | `int` | global | Running count of brushes created (progress display) |
| `q1_numclipbrushes` | `int` | global | Count of brushes discarded due to missing texture info |

## Key Functions / Methods

### Q1_TextureContents
- **Signature:** `int Q1_TextureContents(char *name)`
- **Purpose:** Maps a Q1 texture name to a content constant (`CONTENTS_SOLID`, `CONTENTS_WATER`, `CONTENTS_SLIME`, `CONTENTS_LAVA`).
- **Inputs:** Texture name string.
- **Outputs/Return:** Integer content constant.
- **Side effects:** None.
- **Calls:** `Q_strcasecmp`, `Q_strncasecmp`.
- **Notes:** `"clip"` and `"sky*"` textures map to solid; `"*lava"`, `"*slime"` map to their respective liquids; all others starting with `*` become water.

### Q1_SplitBrush
- **Signature:** `void Q1_SplitBrush(bspbrush_t *brush, int planenum, int nodenum, bspbrush_t **front, bspbrush_t **back)`
- **Purpose:** Splits a brush into front/back halves along a map plane, storing the BSP node number in the new mid-winding's `surf` field so texturing can look up the originating node later.
- **Inputs:** Source brush, plane index, BSP node index, output pointers.
- **Outputs/Return:** `*front` and `*back` filled (either may be NULL).
- **Side effects:** Allocates new `bspbrush_t` via `AllocBrush`; frees degenerate results via `FreeBrush`; logs diagnostic messages.
- **Calls:** `CopyBrush`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingIsTiny`, `WindingIsHuge`, `BrushMostlyOnSide`, `AllocBrush`, `ClipWindingEpsilon`, `BoundBrush`, `FreeBrush`, `CopyWinding`, `BrushVolume`, `Log_Print`.
- **Notes:** Degenerate brushes (volume < 1, bounds outside ±4096, fewer than 3 sides) are freed and set to NULL. The epsilon used is 0 (not `PLANESIDE_EPSILON`) to avoid tiny-winding false positives common in HL maps.

### Q1_CreateBrushes_r
- **Signature:** `bspbrush_t *Q1_CreateBrushes_r(bspbrush_t *brush, int nodenum)`
- **Purpose:** Recursively walks the Q1 BSP tree, splitting the brush at each node plane and tagging leaf fragments with the correct content type.
- **Inputs:** Current brush fragment, current BSP node index (negative = leaf).
- **Outputs/Return:** Linked list of content-tagged `bspbrush_t` fragments.
- **Side effects:** Frees the input brush; may call `Error` on unknown content types; increments `q1_numbrushes`.
- **Calls:** `Q1_SplitBrush`, `FreeBrush`, `FindFloatPlane`, `Log_Print`, `Error`.
- **Notes:** Empty leaves free the brush and return NULL; solid/sky/clip/translucent set `brush->side = CONTENTS_SOLID`; liquid leaves set the appropriate liquid content.

### Q1_CreateBrushesFromBSP
- **Signature:** `bspbrush_t *Q1_CreateBrushesFromBSP(int modelnum)`
- **Purpose:** Bootstraps brush creation by building a single bounding-box brush for the model and passing it to `Q1_CreateBrushes_r`.
- **Inputs:** Model index into `q1_dmodels`.
- **Outputs/Return:** Linked list of all generated brushes.
- **Side effects:** Updates `map_mins`/`map_maxs` via `AddPointToBounds`.
- **Calls:** `BrushFromBounds`, `Q1_CreateBrushes_r`, `AddPointToBounds`, `VectorCopy`.

### Q1_TextureBrushes
- **Signature:** `bspbrush_t *Q1_TextureBrushes(bspbrush_t *brushlist, int modelnum)`
- **Purpose:** Assigns `texinfo` to every brush side by finding the BSP face with the largest overlapping area on the same plane. Optionally splits brushes to fit different textures on adjacent regions.
- **Inputs:** Brush list, model number (0 = world).
- **Outputs/Return:** Possibly restructured brush list (brushes may be split and re-appended).
- **Side effects:** Writes into `map_texinfo[]`; sets `side->flags |= SFL_TEXTURED`; may free and replace brush nodes; calls `Q1_SplitBrushWithFace`.
- **Calls:** `Q1_FaceOnWinding`, `Q1_SplitBrushWithFace`, `FreeBrush`, `FreeBrushList`, `VectorLength`, `Log_Write`, `qprintf`.

### Q1_MergeBrushes
- **Signature:** `bspbrush_t *Q1_MergeBrushes(bspbrush_t *brushlist, int modelnum)`
- **Purpose:** Iteratively tries to merge pairs of same-content brushes until no more merges are possible.
- **Inputs:** Brush list, model number.
- **Outputs/Return:** Reduced brush list.
- **Side effects:** Frees merged brushes; modifies list links.
- **Calls:** `TryMergeBrushes`, `FreeBrush`, `qprintf`.

### Q1_BSPBrushToMapBrush
- **Signature:** `void Q1_BSPBrushToMapBrush(bspbrush_t *bspbrush, entity_t *mapent)`
- **Purpose:** Converts a single `bspbrush_t` into a `mapbrush_t` entry in the global `mapbrushes[]` array, then either hands it to the AAS pipeline or builds windings/bevels for standard map export.
- **Inputs:** Source BSP brush, target map entity.
- **Outputs/Return:** None (modifies global arrays).
- **Side effects:** Increments `nummapbrushes`, `nummapbrushsides`, `mapent->numbrushes`, `q1_numclipbrushes`; calls `AAS_CreateMapBrushes`, `MakeBrushWindings`, `AddBrushBevels`.
- **Calls:** `CheckBSPBrush`, `AAS_CreateMapBrushes`, `MakeBrushWindings`, `AddBrushBevels`, `Error`.

### Q1_LoadMapFromBSP
- **Signature:** `void Q1_LoadMapFromBSP(char *filename, int offset, int length)`
- **Purpose:** Top-level entry point: loads the Q1 BSP file, parses entities, and drives brush creation for every entity with a model reference.
- **Inputs:** BSP filename, file offset, byte length.
- **Outputs/Return:** None.
- **Side effects:** Sets `loadedmaptype = MAPTYPE_QUAKE1`; populates global map arrays; calls `Q1_LoadBSPFile`, `Q1_ParseEntities`, `Q1_CreateMapBrushes`.
- **Calls:** `Q1_LoadBSPFile`, `Q1_ParseEntities`, `ClearBounds`, `Q1_CreateMapBrushes`, `ValueForKey`, `qprintf`, `Log_Print`.

### Notes on minor helpers
- `Q1_SolidTree_r` — recursive predicate checking if all leaves below a node are solid (currently commented out at call site).
- `Q1_PointInLeaf` — BSP traversal returning the leaf containing a point.
- `Q1_FaceArea` — computes polygon area via cross products for fan triangulation.
- `Q1_FacePlane` — derives a plane from the first two edges of a face.
- `Q1_FaceOnWinding` — clips a winding by face edge planes and returns the overlap area.
- `Q1_SplitBrushWithFace` — splits a brush into pieces bounded by a face's edge planes.
- `Q1_FixContentsTextures` — ensures liquid brushes have a matching liquid texture on all sides.
- `Q1_CreateMapBrushes` — per-entity pipeline orchestrator calling create → texture → fix → merge → convert.
- `Q1_ResetMapLoading` — stub; no state to reset.

## Control Flow Notes
This file is part of the offline BSPC tool (not the runtime engine). `Q1_LoadMapFromBSP` is called during tool startup when the input is identified as a Q1/HL BSP. The pipeline runs once, sequentially: load BSP → parse entities → per-entity brush reconstruction → texture assignment → optional merge → export to `mapbrushes[]` / AAS. There is no frame loop or runtime update path.

## External Dependencies
- `qbsp.h` — all core BSP types (`bspbrush_t`, `mapbrush_t`, `side_t`, `plane_t`), global map arrays, and utility function declarations
- `l_bsp_q1.h` — Q1 BSP lump types and loaded BSP data arrays (`q1_dleafs`, `q1_dnodes`, `q1_dfaces`, `q1_dplanes`, etc.)
- `aas_map.h` — `AAS_CreateMapBrushes` (defined in `aas_map.c`)
- **Defined elsewhere:** `FindFloatPlane`, `BrushFromBounds`, `AllocBrush`, `FreeBrush`, `CopyBrush`, `SplitBrush`, `TryMergeBrushes`, `BoundBrush`, `BrushVolume`, `MakeBrushWindings`, `AddBrushBevels`, `CheckBSPBrush`, `BaseWindingForPlane`, `ChopWindingInPlace`, `CopyWinding`, `WindingArea`, `FreeWinding`, `ClipWindingEpsilon`, `Log_Print`, `Q_strcasecmp`, `Q_strncasecmp`, `ValueForKey`, `qprintf`, `Error`, `map_texinfo[]`, `map_mins`/`map_maxs`, `lessbrushes`, `nobrushmerge`, `create_aas`, `nummapbrushes`, `nummapbrushsides`, `entities[]`, `num_entities`
