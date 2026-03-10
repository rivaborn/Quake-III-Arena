# code/bspc/map_q2.c

## File Purpose
Handles loading and parsing of Quake 2 map data for the BSPC (BSP Compiler) tool, supporting both `.map` text format and BSP binary format. It converts Q2 brush/entity data into the internal `mapbrush_t`/`entity_t` representation used by the AAS generation pipeline.

## Core Responsibilities
- Parse Q2 `.map` text files into internal map structures (`Q2_LoadMapFile`)
- Load map geometry directly from compiled Q2 BSP files (`Q2_LoadMapFromBSP`)
- Determine brush contents/surface flags from texture info
- Handle special entity types: `func_group`, `func_areaportal`, origin brushes
- Convert BSP-format brushes (`dbrush_t`) to map brushes (`mapbrush_t`)
- Build per-brush model number mappings by traversing the BSP tree iteratively
- Populate `map_texinfo[]` from loaded Q2 texinfo data

## Key Types / Data Structures
None defined here; all types come from `qbsp.h` (`mapbrush_t`, `side_t`, `entity_t`, `plane_t`, `brush_texture_t`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `nodestack[NODESTACKSIZE]` | `int[1024]` | file-static (`#ifdef ME`) | Manual stack for iterative BSP tree traversal |
| `nodestackptr` | `int *` | file-static | Current top of node stack |
| `nodestacksize` | `int` | file-static | Current depth of node stack |
| `brushmodelnumbers[MAX_MAPFILE_BRUSHES]` | `int[]` | file-static | Maps brush index → BSP model number |
| `dbrushleafnums[MAX_MAPFILE_BRUSHES]` | `int[]` | file-static | Maps brush index → leaf number containing it |
| `dplanes2mapplanes[MAX_MAPFILE_PLANES]` | `int[]` | file-static | Maps BSP plane index → map plane index |

## Key Functions / Methods

### Q2_CreateMapTexinfo
- Signature: `void Q2_CreateMapTexinfo(void)`
- Purpose: Copies loaded BSP/map texinfo into the internal `map_texinfo[]` array.
- Inputs: Global `texinfo[]`, `numtexinfo`
- Outputs/Return: Writes `map_texinfo[]`
- Side effects: Populates global `map_texinfo`
- Calls: `memcpy`, `strcpy`

### Q2_BrushContents
- Signature: `int Q2_BrushContents(mapbrush_t *b)`
- Purpose: Determines unified content flags for a brush by examining all sides; promotes translucent solid brushes to `CONTENTS_WINDOW`.
- Inputs: `mapbrush_t *b`, global `texinfo[]`
- Outputs/Return: Combined `int` contents flags
- Side effects: Logs warnings for mixed-content brushes
- Calls: `Log_Print`

### Q2_ParseBrush
- Signature: `void Q2_ParseBrush(script_t *script, entity_t *mapent)`
- Purpose: Reads one brush block from a `.map` script; resolves planes, texinfo, contents; dispatches to AAS creation or standard processing.
- Inputs: Token stream `script`, owning `mapent`
- Outputs/Return: Populates `mapbrushes[]`, `brushsides[]`
- Side effects: Increments `nummapbrushes`, `nummapbrushsides`; calls `AAS_CreateMapBrushes` if `create_aas`; sets entity origin via `SetKeyValue`
- Calls: `PS_ReadToken`, `PlaneFromPoints`, `FindMiptex`, `TexinfoForBrushTexture`, `Q2_BrushContents`, `MakeBrushWindings`, `AddBrushBevels`, `BrushExists`, `AAS_CreateMapBrushes`
- Notes: Skips degenerate (duplicate/mirrored) planes; handles `nodetail`, `nowater`, origin brush logic

### Q2_MoveBrushesToWorld
- Signature: `void Q2_MoveBrushesToWorld(entity_t *mapent)`
- Purpose: Relocates all brushes from a non-world entity into the world entity's brush list (used for `func_group` and `func_areaportal`).
- Inputs: `entity_t *mapent`
- Outputs/Return: Modifies `mapbrushes[]` in-place; adjusts `firstbrush` indices for all entities
- Side effects: `GetMemory`/`FreeMemory`, `memmove`, `memcpy`

### Q2_ParseMapEntity
- Signature: `qboolean Q2_ParseMapEntity(script_t *script)`
- Purpose: Parses one `{ }` entity block from a `.map` file, collecting key-value pairs and brushes; handles origin offsets and special entity classes.
- Inputs: Token stream `script`
- Outputs/Return: `qboolean` (false on EOF)
- Side effects: Increments `num_entities`; may call `Q2_MoveBrushesToWorld`
- Calls: `Q2_ParseBrush`, `ParseEpair`, `GetVectorForKey`, `TexinfoForBrushTexture`, `MakeBrushWindings`, `Q2_MoveBrushesToWorld`

### Q2_LoadMapFile
- Signature: `void Q2_LoadMapFile(char *filename)`
- Purpose: Top-level entry point to load a Q2 `.map` text file.
- Inputs: `filename`
- Outputs/Return: Populates all global map arrays
- Side effects: Sets `loadedmaptype = MAPTYPE_QUAKE2`; calls `ResetMapLoading`, `PrintMapInfo`, `Q2_CreateMapTexinfo`
- Calls: `LoadScriptFile`, `Q2_ParseMapEntity`, `ClearBounds`, `AddPointToBounds`, `FreeScript`

### Q2_BSPBrushToMapBrush
- Signature: `void Q2_BSPBrushToMapBrush(dbrush_t *bspbrush, entity_t *mapent)`
- Purpose: Converts a single BSP-format brush into an internal `mapbrush_t`, resolving planes and surface flags from binary BSP data.
- Inputs: `dbrush_t *bspbrush`, `entity_t *mapent`
- Side effects: Increments `nummapbrushes`, `nummapbrushsides`; may call `AAS_CreateMapBrushes`
- Calls: `FindFloatPlane`, `Q2_BrushContents`, `BrushExists`, `MakeBrushWindings`, `MarkBrushBevels`, `AAS_CreateMapBrushes`

### Q2_SetBrushModelNumbers
- Signature: `void Q2_SetBrushModelNumbers(entity_t *mapent)`
- Purpose: Iterates the BSP tree rooted at the entity's head node to assign `brushmodelnumbers[]` and `dbrushleafnums[]` entries using an explicit stack instead of recursion.
- Inputs: `entity_t *mapent` (uses `mapent->modelnum`)
- Side effects: Writes `brushmodelnumbers[]`, `dbrushleafnums[]`
- Calls: `Q2_InitNodeStack`, `Q2_PushNodeStack`, `Q2_PopNodeStack`, `Q2_SetLeafBrushesModelNumbers`

### Q2_LoadMapFromBSP
- Signature: `void Q2_LoadMapFromBSP(char *filename, int offset, int length)`
- Purpose: Top-level entry point to reconstruct map geometry from a compiled Q2 BSP file.
- Side effects: Calls `Q2_LoadBSPFile`, `Q2_ParseEntities`, `Q2_ParseBSPEntity` per entity
- Calls: `Q2_LoadBSPFile`, `Q2_ParseEntities`, `Q2_ParseBSPEntity`, `ClearBounds`, `AddPointToBounds`, `PrintMapInfo`, `Q2_CreateMapTexinfo`

### TestExpandBrushes
- Signature: `void TestExpandBrushes(void)`
- Purpose: Debug utility — expands all brush planes outward by 16 units and writes a `.map` file, then aborts.
- Side effects: File I/O, calls `Error` unconditionally at end

## Control Flow Notes
- **Init**: Called from `LoadMapFromBSP` (in `map.c`) or directly as `Q2_LoadMapFile` / `Q2_LoadMapFromBSP`.
- **No frame/update involvement**; this is an offline tool (BSPC), not a runtime game system.
- The BSP-loading path (`Q2_LoadMapFromBSP`) replaces text parsing with binary BSP data, traversing the BSP tree iteratively via a manual node stack to assign brush-to-model mappings before converting brushes.
- If `create_aas` is set, brushes are forwarded to `AAS_CreateMapBrushes` early and the normal pipeline is skipped.

## External Dependencies
- `qbsp.h` — all core types and globals (`mapbrushes`, `brushsides`, `mapplanes`, `entities`, etc.)
- `l_mem.h` — `GetMemory`, `FreeMemory`
- `botlib/aasfile.h` — `aas_bbox_t` (included for AAS type sizes)
- `aas_store.h` — `AAS_MAX_BBOXES`
- `aas_cfg.h` — `cfg_t` / AAS physics config
- `aas_map.h` — `AAS_CreateMapBrushes` (defined elsewhere)
- `l_bsp_q2.h` — `Q2_LoadBSPFile`, `Q2_ParseEntities`, BSP lump globals (`dbrushes`, `dbrushsides`, `dleafs`, `dleafbrushes`, `dfaces`, `texinfo`, etc.) — defined elsewhere
