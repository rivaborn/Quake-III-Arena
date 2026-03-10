# code/bspc/map_sin.c

## File Purpose
Converts Sin game BSP data into the BSPC tool's internal map brush representation for AAS (Area Awareness System) generation. It handles both direct `.map` file parsing (commented out) and BSP-file-to-mapbrush conversion paths, including Sin-specific content/surface flag semantics.

## Core Responsibilities
- Determine brush content flags from side surface data (`Sin_BrushContents`)
- Initialize a default `map_texinfo` entry and copy Sin BSP texinfo data (`Sin_CreateMapTexinfo`)
- Traverse the BSP tree iteratively to assign model numbers to leaf brushes (`Sin_SetBrushModelNumbers` via node stack helpers)
- Convert individual `sin_dbrush_t` BSP brushes into `mapbrush_t` map brushes (`Sin_BSPBrushToMapBrush`)
- Parse all BSP brushes belonging to a given entity (`Sin_ParseBSPBrushes`, `Sin_ParseBSPEntity`)
- Drive the full BSP-to-map load pipeline (`Sin_LoadMapFromBSP`)
- Reset loader state between runs (`Sin_ResetMapLoading`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `mapbrush_t` | struct (typedef) | Internal map brush: entity/brush numbers, contents, bounds, sides array |
| `side_t` | struct (typedef) | One face of a brush: planenum, texinfo, winding, contents, surf flags |
| `sin_dbrush_t` | struct (extern) | Raw Sin BSP brush record (firstside, numsides, contents) |
| `sin_dbrushside_t` | struct (extern) | Raw Sin BSP brush side (planenum, texinfo) |
| `sin_dleaf_t` | struct (extern) | BSP leaf containing leaf-brush index lists |
| `map_texinfo_t` | struct (typedef) | Internal texinfo: vec arrays, flags, value, texture name, chain |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `nodestack` | `int[NODESTACKSIZE]` | global | Manual stack for iterative BSP tree traversal |
| `nodestackptr` | `int *` | global | Current top pointer into `nodestack` |
| `nodestacksize` | `int` | global | Current depth of node stack |
| `brushmodelnumbers` | `int[MAX_MAPFILE_BRUSHES]` | global | Maps brush index → BSP model number |
| `dbrushleafnums` | `int[MAX_MAPFILE_BRUSHES]` | global | Maps brush index → BSP leaf number |
| `nummapbrushes` | `int` | global | Running count of converted map brushes |
| `nummapbrushsides` | `int` | global | Running count of converted brush sides |
| `loadedmaptype` | `int` | global | Tracks which game format was loaded (`MAPTYPE_SIN`) |

## Key Functions / Methods

### Sin_BrushContents
- **Signature:** `int Sin_BrushContents(mapbrush_t *b)`
- **Purpose:** Derives aggregate content flags for a brush from all its sides' texinfo data, resolving DETAIL/FENCE/TRANSLUCENT conflicts.
- **Inputs:** `b` — pointer to the map brush to evaluate.
- **Outputs/Return:** `int` — combined content flags bitmask.
- **Side effects:** Prints warnings to stdout on mixed content.
- **Calls:** `printf`
- **Notes:** Under `#ifdef SIN`, translucence is summed (float); without SIN, flags are OR'd. FENCE content is remapped to WINDOW+DUMMYFENCE+DETAIL.

### Sin_CreateMapTexinfo
- **Signature:** `void Sin_CreateMapTexinfo(void)`
- **Purpose:** Populates `map_texinfo[]` from `sin_texinfo[]`, setting entry 0 to a default red-texture fallback.
- **Inputs:** None (reads global `sin_texinfo`, `sin_numtexinfo`).
- **Outputs/Return:** None.
- **Side effects:** Writes `map_texinfo[0..sin_numtexinfo-1]`.
- **Calls:** `memcpy`, `strcpy`

### Sin_SetBrushModelNumbers
- **Signature:** `void Sin_SetBrushModelNumbers(entity_t *mapent)`
- **Purpose:** Iterative DFS of the Sin BSP tree rooted at `mapent`'s model headnode; calls `Sin_SetLeafBrushesModelNumbers` at each leaf.
- **Inputs:** `mapent` — entity whose BSP model tree is traversed.
- **Outputs/Return:** None.
- **Side effects:** Sets `brushmodelnumbers[]` and `dbrushleafnums[]` for all brushes reachable from the entity's model.
- **Calls:** `Sin_InitNodeStack`, `Sin_PushNodeStack`, `Sin_PopNodeStack`, `Sin_SetLeafBrushesModelNumbers`

### Sin_BSPBrushToMapBrush
- **Signature:** `void Sin_BSPBrushToMapBrush(sin_dbrush_t *bspbrush, entity_t *mapent)`
- **Purpose:** Converts one raw BSP brush record and its sides into a `mapbrush_t`, applying content classification, duplicate plane detection, winding generation, and optional AAS brush creation.
- **Inputs:** `bspbrush` — source BSP brush; `mapent` — owning entity.
- **Outputs/Return:** None (appends to `mapbrushes[]`, increments counters).
- **Side effects:** Modifies `nummapbrushes`, `nummapbrushsides`, `brushsides[]`, `mapbrushes[]`; may call `AAS_CreateMapBrushes`; increments `c_clipbrushes`, `c_squattbrushes`.
- **Calls:** `Sin_BrushContents`, `BrushExists`, `AAS_CreateMapBrushes`, `MakeBrushWindings`, `MarkBrushBevels`, `FindFloatPlane`, `Log_Print`

### Sin_LoadMapFromBSP
- **Signature:** `void Sin_LoadMapFromBSP(char *filename, int offset, int length)`
- **Purpose:** Top-level entry point — loads the Sin BSP file, parses all entities, converts all brushes, and builds `map_texinfo`.
- **Inputs:** `filename`, `offset`, `length` — BSP file location.
- **Outputs/Return:** None.
- **Side effects:** Sets `loadedmaptype = MAPTYPE_SIN`; populates all global map arrays; computes `map_mins`/`map_maxs`.
- **Calls:** `Sin_LoadBSPFile`, `Sin_ParseEntities`, `Sin_ParseBSPEntity`, `Sin_CreateMapTexinfo`, `ClearBounds`, `AddPointToBounds`, `Log_Print`

- **Notes on helpers:** `Sin_InitNodeStack`/`Sin_PushNodeStack`/`Sin_PopNodeStack` manage a simple fixed-size integer stack. `Sin_SetLeafBrushesModelNumbers` maps leaf-brush indices to a model number. `Sin_ResetMapLoading` zeros all stack and model-number state.

## Control Flow Notes
`Sin_LoadMapFromBSP` is the entry point called during BSP conversion. It drives: BSP file load → entity string parse → per-entity brush parse (`Sin_ParseBSPEntity` → `Sin_ParseBSPBrushes` → `Sin_BSPBrushToMapBrush`) → texinfo table build. The `#ifdef ME` guard wraps the entire BSP-from-file path; the `.map` text-parsing functions (`ParseBrush`, `MoveBrushesToWorld`, `Sin_ParseMapEntity`, `Sin_LoadMapFile`) are commented out with `/* … */`.

## External Dependencies
- `qbsp.h` — all core types (`mapbrush_t`, `side_t`, `plane_t`, global arrays, constants)
- `l_bsp_sin.h` / `sinfiles.h` — Sin BSP data arrays (`sin_dbrushes`, `sin_texinfo`, `sin_dleafs`, etc.)
- `aas_map.h` — `AAS_CreateMapBrushes` (defined in `aas_map.c`)
- **Defined elsewhere:** `FindFloatPlane`, `MakeBrushWindings`, `MarkBrushBevels`, `BrushExists`, `AddBrushBevels`, `Log_Print`, `Sin_LoadBSPFile`, `Sin_ParseEntities`, `ValueForKey`, `GetVectorForKey`
