# code/bspc/map.c

## File Purpose
This file is the central map data manager for the BSPC (BSP Compiler) tool. It handles plane management, brush geometry construction, and multi-format BSP map loading/writing. It serves as the unified interface for converting BSP files from various Quake-engine games into a normalized internal map representation.

## Core Responsibilities
- Maintain global arrays of map planes, brushes, and brush sides
- Find or create float planes with hash-based deduplication
- Add axial and edge bevel sides to brushes for AAS expansion
- Generate brush side windings and bounding boxes
- Write map data back to `.map` text files (multi-format aware)
- Dispatch BSP loading to the correct format handler (Q1/Q2/Q3/HL/SIN)
- Reset all map state between load operations

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `idheader_t` | struct | Minimal BSP file header (ident + version) for format detection |
| `plane_t` | struct (from qbsp.h) | Map plane with normal, dist, type, signbits, hash chain |
| `mapbrush_t` | struct (from qbsp.h) | A map brush with sides, bounds, entity/brush number |
| `side_t` | struct (from qbsp.h) | A brush side with planenum, texinfo, winding, flags |
| `map_texinfo_t` | struct (from qbsp.h) | Texture projection vectors, flags, value, texture name |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `nummapbrushes` | `int` | global | Count of loaded map brushes |
| `mapbrushes[MAX_MAPFILE_BRUSHES]` | `mapbrush_t[]` | global | All map brushes |
| `nummapbrushsides` | `int` | global | Count of brush sides |
| `brushsides[MAX_MAPFILE_BRUSHSIDES]` | `side_t[]` | global | All brush sides |
| `side_brushtextures[...]` | `brush_texture_t[]` | global | Texture data per brush side |
| `nummapplanes` | `int` | global | Count of map planes |
| `mapplanes[MAX_MAPFILE_PLANES]` | `plane_t[]` | global | All map planes |
| `mapplaneusers[MAX_MAPFILE_PLANES]` | `int[]` | global | Reference counts per plane |
| `planehash[PLANE_HASHES]` | `plane_t*[]` | global | Hash table for fast plane lookup |
| `map_mins`, `map_maxs` | `vec3_t` | global | World bounding box |
| `map_texinfo[MAX_MAPFILE_TEXINFO]` | `map_texinfo_t[]` | global | All texture infos |
| `map_numtexinfo` | `int` | global | Count of texture infos |
| `loadedmaptype` | `int` | global | Enum indicating format (Q1/Q2/Q3/HL/SIN) |
| `c_boxbevels`, `c_edgebevels` | `int` | global | Bevel counters for stats |
| `c_areaportals`, `c_clipbrushes`, `c_squattbrushes`, `c_writtenbrushes` | `int` | global | Brush processing counters |

## Key Functions / Methods

### FindFloatPlane
- Signature: `int FindFloatPlane(vec3_t normal, vec_t dist)`
- Purpose: Returns index of an existing matching plane or creates a new one. Uses hash bucketing for O(1) average lookup.
- Inputs: Plane normal vector, distance from origin
- Outputs/Return: Index into `mapplanes[]`
- Side effects: May extend `mapplanes[]` and `planehash[]`; increments `mapplaneusers[]`
- Calls: `SnapPlane`, `PlaneEqual`, `CreateNewFloatPlane`
- Notes: Two variants — linear scan (`#ifndef USE_HASHING`) and hash (`#else`); hash searches ±1 neighboring bins for epsilon tolerance

### CreateNewFloatPlane
- Signature: `int CreateNewFloatPlane(vec3_t normal, vec_t dist)`
- Purpose: Allocates a new plane pair (plane + its opposite) in `mapplanes[]`
- Inputs: Normal, distance
- Outputs/Return: Index of the forward plane
- Side effects: Writes two entries into `mapplanes[]`, increments `nummapplanes` by 2, calls `AddPlaneToHash` twice
- Notes: Axial planes with negative normals are swapped to ensure positive-facing planes appear first

### AddBrushBevels
- Signature: `void AddBrushBevels(mapbrush_t *b)`
- Purpose: Adds axial bevel planes and non-axial edge bevel planes needed to correctly expand the brush against AABB bounds
- Inputs: Pointer to the brush to process
- Outputs/Return: void
- Side effects: Extends `b->original_sides`, increments `nummapbrushsides`, `c_boxbevels`, `c_edgebevels`; reorders sides to canonical order
- Calls: `FindFloatPlane`, `VectorNormalize`, `CrossProduct`, `DotProduct`, `PlaneEqual`
- Notes: Early-out if brush already has exactly 6 sides (pure axial); processes all non-axial winding edges for up to 6 slanted bevels each

### MakeBrushWindings
- Signature: `qboolean MakeBrushWindings(mapbrush_t *ob)`
- Purpose: Computes a clipped winding for each brush side and calculates the brush's AABB
- Inputs: Map brush
- Outputs/Return: `true` always, but sets `ob->numsides = 0` on out-of-bounds brushes
- Side effects: Allocates `winding_t` objects into `side_t::winding` fields; modifies `ob->mins/maxs`
- Calls: `BaseWindingForPlane`, `ChopWindingInPlace`, `AddPointToBounds`, `ClearBounds`, `Log_Print`
- Notes: Skips bevel sides when clipping; marks sides `SFL_VISIBLE` if they have a valid winding

### MarkBrushBevels
- Signature: `void MarkBrushBevels(mapbrush_t *brush)`
- Purpose: Post-hoc identification of degenerate sides (no winding, tiny winding, winding errors) and tags them `SFL_BEVEL`
- Calls: `WindingIsTiny`, `WindingError`, `WindingErrorString`, `Log_Write`
- Notes: Does not fully mark all bevels per the FIXME comment; increments `c_boxbevels`/`c_edgebevels` for axial vs. non-axial degenerate planes

### WriteMapFile / WriteMapFileSafe
- Signature: `void WriteMapFile(char *filename)` / `qboolean WriteMapFileSafe(FILE *fp)`
- Purpose: Serializes the loaded map back to `.map` text format, routing texture/content names through per-format logic
- Inputs: Output filename or file pointer
- Outputs/Return: void / qboolean success
- Side effects: File I/O; increments `c_writtenbrushes`
- Calls: `WriteMapBrush`, `WriteOriginBrush`, `GetAreaPortalBrush`, `ValueForKey`, `SetKeyValue`, `GetVectorForKey`, `StripTrailing`, `Log_Print`

### LoadMapFromBSP
- Signature: `int LoadMapFromBSP(struct quakefile_s *qf)`
- Purpose: Detects BSP format from the file header and dispatches to the appropriate format-specific loader
- Inputs: `quakefile_s` with filename, offset, length
- Outputs/Return: 1 on success, 0 on failure
- Side effects: Calls `ResetMapLoading` then the appropriate Alloc/Load/Free trio; populates all global map arrays
- Calls: `ResetMapLoading`, `Q3_LoadMapFromBSP`, `Q2_AllocMaxBSP`, `Q2_LoadMapFromBSP`, `Q2_FreeMaxBSP`, `Sin_*`, `Q1_*`, `HL_*`, `LittleLong`, `Error`
- Notes: Supports Q1 (v29), Q2 (IBSP v38), Q3 (IBSP v46), SIN (IBSP v41 / RBSP v1), Half-Life (v30)

### ResetMapLoading
- Signature: `void ResetMapLoading(void)`
- Purpose: Frees all winding memory and zeroes all global map state before loading a new map
- Side effects: Calls `Q2_ResetMapLoading`, `Sin_ResetMapLoading`, `FreeWinding`, `FreeMemory` on all entity epairs; zeroes all global arrays

## Control Flow Notes
This file is a **tool-time** (offline) module, not runtime game code. It has no frame loop. The typical call sequence is:
1. `LoadMapFromBSP` → `ResetMapLoading` → format-specific loader populates global arrays
2. Callers invoke `MakeBrushWindings` and `AddBrushBevels` per brush to build geometry
3. `MarkBrushBevels` further tags degenerate sides
4. `WriteMapFile` serializes back to text, or the AAS pipeline proceeds to cluster/route generation

## External Dependencies
- `qbsp.h` — core types (`plane_t`, `mapbrush_t`, `side_t`, `winding_t`, `entity_t`, `face_t`), math macros, all subsystem prototypes
- `l_bsp_hl.h`, `l_bsp_q1.h`, `l_bsp_q2.h`, `l_bsp_q3.h`, `l_bsp_sin.h` — per-format BSP loader interfaces
- `l_mem.h` — `FreeMemory`, `FreeWinding`, `BaseWindingForPlane`, `ChopWindingInPlace`
- `aasfile.h` / `aas_store.h` / `aas_cfg.h` — AAS bounding box constants (included, not directly used in visible code)
- Defined elsewhere: `entities[]`, `num_entities`, `epair_t`, `entity_t`, `ReadQuakeFile`, `I_FloatTime`, `TextureAxisFromPlane`, `VectorNormalize2`, `Log_Write`, `Log_Print`, `Error`
