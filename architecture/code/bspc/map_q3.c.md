# code/bspc/map_q3.c

## File Purpose
Converts a compiled Quake III BSP file into the internal `mapbrush_t` representation used by the BSPC tool. It handles both solid brushes and curved patch surfaces, translating BSP-native structures into map geometry suitable for AAS generation or further BSP processing.

## Core Responsibilities
- Determine consolidated content flags for a map brush from its sides (`Q3_BrushContents`)
- Register BSP dplane entries into the map-plane table (`Q3_DPlanes2MapPlanes`)
- Convert a single `q3_dbrush_t` BSP brush into a `mapbrush_t` (`Q3_BSPBrushToMapBrush`)
- Parse all brushes belonging to a BSP model/entity (`Q3_ParseBSPBrushes`, `Q3_ParseBSPEntity`)
- Tessellate Q3 patch surfaces into convex brush proxies for collision/AAS (`AAS_CreateCurveBrushes`)
- Orchestrate full BSP-to-map load pipeline and compute world AABB (`Q3_LoadMapFromBSP`)
- Reset transient state between map loads (`Q3_ResetMapLoading`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `mapbrush_t` | struct (typedef) | Internal brush representation with sides, bounds, entity/brush indices |
| `side_t` | struct (typedef) | One face of a brush; carries plane number, surface/content flags, winding |
| `q3_dbrush_t` | struct (defined elsewhere) | Raw BSP brush lump entry |
| `q3_dbrushside_t` | struct (defined elsewhere) | Raw BSP brush-side lump entry |
| `patchCollide_t` / `facet_t` | structs (defined in cm_patch) | Intermediate patch collision geometry output by `CM_GeneratePatchCollide` |
| `entity_t` | struct (defined elsewhere) | Map entity owning a set of brushes and a BSP model |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `nummapbrushes` | `int` | global (extern in qbsp.h) | Running count of all converted map brushes |
| `nummapbrushsides` | `int` | global (extern in qbsp.h) | Running count of all brush sides |
| `mapbrushes[]` | `mapbrush_t[MAX_MAPFILE_BRUSHES]` | global | Flat array storing all converted brushes |
| `brushsides[]` | `side_t[MAX_MAPFILE_BRUSHSIDES]` | global | Flat array storing all brush sides |
| `mapplanes[]` | `plane_t[MAX_MAPFILE_PLANES]` | global | Map plane table; indexed by `side_t::planenum` |
| `loadedmaptype` | `int` | global | Set to `MAPTYPE_QUAKE3` on load |
| `nodestack`, `nodestackptr`, `nodestacksize` | `int[]`, `int*`, `int` | global | BSP traversal stack; reset by `Q3_ResetMapLoading` |
| `brushmodelnumbers[]` | `int[MAX_MAPFILE_BRUSHES]` | global | Maps brush index → BSP model number; reset on load |
| `c_squattbrushes`, `c_clipbrushes`, `c_areaportals` | `int` | global | Diagnostic counters |
| `map_mins`, `map_maxs` | `vec3_t` | global | World-space bounding box computed from brush bounds |
| `create_aas` | `qboolean` | global (extern) | When true, routes brushes to AAS creation instead of BSP |

## Key Functions / Methods

### Q3_BrushContents
- **Signature:** `int Q3_BrushContents(mapbrush_t *b)`
- **Purpose:** Computes a single consolidated content value for the brush by OR-ing all sides' content flags, then resolving priority among mixed types.
- **Inputs:** `b` — pointer to a partially-initialised `mapbrush_t` whose `original_sides` are already populated.
- **Outputs/Return:** Integer content flags (e.g. `CONTENTS_SOLID`, `CONTENTS_LAVA`). Returns `0` for hint/skip brushes or structural-only brushes.
- **Side effects:** Calls `Log_Write` / `Log_Print` for diagnostics on mixed or hint content.
- **Calls:** `Log_Write`, `Log_Print`, `PrintContents`
- **Notes:** Strips `CONTENTS_LADDER` and `CONTENTS_FOG` unconditionally. Priority order on mixed: `DONOTENTER` > liquid > `PLAYERCLIP` > `SOLID`.

### Q3_DPlanes2MapPlanes
- **Signature:** `void Q3_DPlanes2MapPlanes(void)`
- **Purpose:** Registers every BSP dplane into the BSPC map-plane hash table, filling `dplanes2mapplanes[]`.
- **Inputs:** None (reads global `q3_dplanes`, `q3_numplanes`).
- **Outputs/Return:** void; populates `dplanes2mapplanes[i]`.
- **Side effects:** May extend `mapplanes[]` / `nummapplanes` via `FindFloatPlane`.
- **Calls:** `FindFloatPlane`

### Q3_BSPBrushToMapBrush
- **Signature:** `void Q3_BSPBrushToMapBrush(q3_dbrush_t *bspbrush, entity_t *mapent)`
- **Purpose:** Converts one BSP brush lump entry into a `mapbrush_t`, resolving planes, shader flags, and duplicate/mirror plane detection.
- **Inputs:** `bspbrush` — BSP brush; `mapent` — owning entity.
- **Outputs/Return:** void; appends to `mapbrushes[]` / `brushsides[]`.
- **Side effects:** Increments `nummapbrushes`, `nummapbrushsides`, `mapent->numbrushes`; may route to `AAS_CreateMapBrushes`; calls `MakeBrushWindings`, `MarkBrushBevels`.
- **Calls:** `FindFloatPlane`, `strstr`, `BrushExists`, `AAS_CreateMapBrushes`, `MakeBrushWindings`, `MarkBrushBevels`, `Log_Print`, `Error`
- **Notes:** Duplicate and mirrored planes are detected and skipped (logged). Origin brush re-centering is intentionally disabled (already handled by BSP). Bevel addition is also disabled for the same reason.

### Q3_ParseBSPBrushes
- **Signature:** `void Q3_ParseBSPBrushes(entity_t *mapent)`
- **Purpose:** Iterates the BSP model's brush list and converts each brush via `Q3_BSPBrushToMapBrush`.
- **Inputs:** `mapent` with a valid `modelnum`.
- **Outputs/Return:** void.
- **Side effects:** Delegates entirely to `Q3_BSPBrushToMapBrush`.
- **Calls:** `Q3_BSPBrushToMapBrush`

### Q3_ParseBSPEntity
- **Signature:** `qboolean Q3_ParseBSPEntity(int entnum)`
- **Purpose:** Initialises one entity's map-brush state from BSP model data; counts area portals.
- **Inputs:** `entnum` — index into global `entities[]`.
- **Outputs/Return:** Always `true`.
- **Side effects:** Sets `mapent->modelnum`, `mapent->origin`; calls `Q3_ParseBSPBrushes` when model is present; increments `c_areaportals`.
- **Calls:** `ValueForKey`, `GetVectorForKey`, `atoi`, `Q3_ParseBSPBrushes`, `strcmp`

### AAS_CreateCurveBrushes
- **Signature:** `void AAS_CreateCurveBrushes(void)`
- **Purpose:** Tessellates all solid patch surfaces in the BSP into synthetic convex brush proxies for AAS/collision use.
- **Inputs:** None (reads `q3_drawSurfaces`, `q3_drawVerts`, globals).
- **Outputs/Return:** void; appends brushes to `mapbrushes[]`.
- **Side effects:** Calls `CM_GeneratePatchCollide` per patch, then `AAS_CreateMapBrushes` or `MakeBrushWindings`+`AddBrushBevels` depending on `create_aas`; increments `nummapbrushes`.
- **Calls:** `CM_GeneratePatchCollide`, `FindFloatPlane`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `AAS_CreateMapBrushes`, `MakeBrushWindings`, `AddBrushBevels`, `Log_Print`, `qprintf`, `Error`
- **Notes:** Skips non-solid patches and patches flagged `CONTENTS_NOBOTCLIP`. Validates brush bounds against `MAX_MAP_BOUNDS`.

### Q3_LoadMapFromBSP
- **Signature:** `void Q3_LoadMapFromBSP(struct quakefile_s *qf)`
- **Purpose:** Top-level entry point: loads the BSP, parses entities, converts brushes, creates curve brushes, and computes the world AABB.
- **Inputs:** `qf` — quake file descriptor (path + offset/length).
- **Outputs/Return:** void.
- **Side effects:** Sets `loadedmaptype`; resets `nummapbrushsides`, `num_entities`; populates all map globals.
- **Calls:** `Q3_LoadBSPFile`, `Q3_ParseEntities`, `Q3_ParseBSPEntity`, `AAS_CreateCurveBrushes`, `ClearBounds`, `AddPointToBounds`, `Log_Print`

### Q3_ResetMapLoading
- **Signature:** `void Q3_ResetMapLoading(void)`
- **Purpose:** Clears node-stack and brush-model-number arrays to prepare for a fresh load.
- **Inputs/Outputs:** None.
- **Side effects:** Zeroes `nodestack[]`, `brushmodelnumbers[]`; resets `nodestackptr`, `nodestacksize`.
- **Calls:** `memset`

## Control Flow Notes

This file is a **load-time** module, executed once per BSP-to-map conversion run. The call sequence is:

1. `Q3_ResetMapLoading` — zeroes transient state.
2. `Q3_LoadMapFromBSP` — top-level orchestrator called from `LoadMapFromBSP` (map.c).
   - `Q3_LoadBSPFile` → raw BSP lump data in globals.
   - `Q3_ParseEntities` → populates `entities[]`.
   - Per-entity: `Q3_ParseBSPEntity` → `Q3_ParseBSPBrushes` → `Q3_BSPBrushToMapBrush`.
   - `AAS_CreateCurveBrushes` — patch geometry pass.
   - World AABB computed from `mapbrushes[]`.

There is no per-frame involvement; this is a batch offline tool pipeline.

## External Dependencies

- **`qbsp.h`** — master include: map globals, `plane_t`, `mapbrush_t`, `side_t`, `entity_t`, utility prototypes.
- **`l_bsp_q3.h`** — Q3 BSP lump types (`q3_dbrush_t`, `q3_dbrushside_t`, `q3_dshaders`, `q3_drawSurfaces`, etc.) and `Q3_LoadBSPFile`, `Q3_ParseEntities`.
- **`cm_patch.h`** — `CM_GeneratePatchCollide`, `patchCollide_t`, `facet_t`.
- **`aas_map.h`** — `AAS_CreateMapBrushes`.
- **`surfaceflags.h`** — `SURF_HINT`, `SURF_SKIP`, `SURF_NODRAW`, `CONTENTS_*` constants.
- **Defined elsewhere:** `FindFloatPlane`, `MakeBrushWindings`, `MarkBrushBevels`, `AddBrushBevels`, `BrushExists`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingBounds`, `ClearBounds`, `AddPointToBounds`, `PrintContents`, `Log_Write`, `Log_Print`, `qprintf`, `Error`.
