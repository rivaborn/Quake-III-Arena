# code/bspc/l_bsp_q3.h

## File Purpose
This header declares the global BSP data arrays and counts for a loaded Quake III Arena `.bsp` file, as well as the three public functions used to load, free, and parse that data. It serves as the interface between the BSPC tool's Q3-format BSP reader (`l_bsp_q3.c`) and the rest of the BSPC compiler pipeline.

## Core Responsibilities
- Expose all Q3 BSP lump data arrays as `extern` globals for cross-translation-unit access
- Expose corresponding element-count integers for each lump array
- Declare the three entry-point functions: load, free, and entity-parse

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `q3_dmodel_t` | struct (from q3files.h) | Submodel bounds and surface/brush index ranges |
| `q3_dshader_t` | struct | Shader name + surface/content flags per shader reference |
| `q3_dleaf_t` | struct | BSP leaf: cluster, area, AABB, leaf-surface and leaf-brush index ranges |
| `q3_dplane_t` | struct | BSP splitting plane (normal + dist) |
| `q3_dnode_t` | struct | BSP interior node: plane, children, AABB |
| `q3_dbrush_t` | struct | Brush: first side, side count, shader reference |
| `q3_dbrushside_t` | struct | One side of a brush: plane index + shader index |
| `q3_drawVert_t` | struct | Draw vertex: position, UVs, normal, color |
| `q3_dsurface_t` | struct | Draw surface: shader, fog, type, vert/index ranges, lightmap metadata |
| `q3_dfog_t` | struct | Fog volume: shader, brush, visible side |
| `q3_mapSurfaceType_t` | enum | Surface geometry class (planar, patch, triangle soup, flare) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `q3_nummodels` / `q3_dmodels` | `int` / `q3_dmodel_t *` | global (extern) | Submodel count and array |
| `q3_numShaders` / `q3_dshaders` | `int` / `q3_dshader_t *` | global (extern) | Shader reference count and array |
| `q3_entdatasize` / `q3_dentdata` | `int` / `char *` | global (extern) | Entity lump byte count and raw text |
| `q3_numleafs` / `q3_dleafs` | `int` / `q3_dleaf_t *` | global (extern) | Leaf count and array |
| `q3_numplanes` / `q3_dplanes` | `int` / `q3_dplane_t *` | global (extern) | Plane count and array |
| `q3_numnodes` / `q3_dnodes` | `int` / `q3_dnode_t *` | global (extern) | Node count and array |
| `q3_numleafsurfaces` / `q3_dleafsurfaces` | `int` / `int *` | global (extern) | Leaf-to-surface index list |
| `q3_numleafbrushes` / `q3_dleafbrushes` | `int` / `int *` | global (extern) | Leaf-to-brush index list |
| `q3_numbrushes` / `q3_dbrushes` | `int` / `q3_dbrush_t *` | global (extern) | Brush count and array |
| `q3_numbrushsides` / `q3_dbrushsides` | `int` / `q3_dbrushside_t *` | global (extern) | Brush side count and array |
| `q3_numLightBytes` / `q3_lightBytes` | `int` / `byte *` | global (extern) | Lightmap lump size and raw data |
| `q3_numGridPoints` / `q3_gridData` | `int` / `byte *` | global (extern) | Light grid point count and raw data |
| `q3_numVisBytes` / `q3_visBytes` | `int` / `byte *` | global (extern) | PVS data byte count and raw data |
| `q3_numDrawVerts` / `q3_drawVerts` | `int` / `q3_drawVert_t *` | global (extern) | Draw vertex count and array |
| `q3_numDrawIndexes` / `q3_drawIndexes` | `int` / `int *` | global (extern) | Draw index count and array |
| `q3_numDrawSurfaces` / `q3_drawSurfaces` | `int` / `q3_dsurface_t *` | global (extern) | Draw surface count and array |
| `q3_numFogs` / `q3_dfogs` | `int` / `q3_dfog_t *` | global (extern) | Fog volume count and array |
| `q3_dbrushsidetextured` | `char[Q3_MAX_MAP_BRUSHSIDES]` | global (extern) | Per-brush-side boolean: has texture been assigned |

## Key Functions / Methods

### Q3_LoadBSPFile
- **Signature:** `void Q3_LoadBSPFile(struct quakefile_s *qf)`
- **Purpose:** Reads and deserializes all BSP lumps from a Q3 `.bsp` file into the global arrays above.
- **Inputs:** `qf` — pointer to a `quakefile_s` describing the file path/pak location.
- **Outputs/Return:** `void`; populates all `q3_num*` counts and `q3_d*` pointer arrays.
- **Side effects:** Allocates heap memory for each lump array; sets all global state declared in this header.
- **Calls:** Not inferable from this file alone (defined in `l_bsp_q3.c`).
- **Notes:** Must be called before any other system that reads the `q3_d*` globals.

### Q3_FreeMaxBSP
- **Signature:** `void Q3_FreeMaxBSP(void)`
- **Purpose:** Releases all heap memory allocated by `Q3_LoadBSPFile` and resets counts to zero.
- **Inputs:** None.
- **Outputs/Return:** `void`
- **Side effects:** Frees all `q3_d*` arrays; zeros `q3_num*` counts.
- **Calls:** Not inferable from this file alone.
- **Notes:** Counterpart to `Q3_LoadBSPFile`; should be called during shutdown or before loading a new map.

### Q3_ParseEntities
- **Signature:** `void Q3_ParseEntities(void)`
- **Purpose:** Parses the raw entity-lump text (`q3_dentdata`) into structured entity key/value pairs for use by the BSPC pipeline.
- **Inputs:** None (reads from `q3_dentdata` / `q3_entdatasize` globals).
- **Outputs/Return:** `void`; results stored in shared entity list (defined elsewhere).
- **Side effects:** Modifies global entity list; depends on `Q3_LoadBSPFile` having been called first.
- **Calls:** Not inferable from this file alone.

## Control Flow Notes
This header is consumed during the BSPC tool's map-loading phase. The typical flow is: `Q3_LoadBSPFile` → `Q3_ParseEntities` → BSP-to-AAS conversion routines access the `q3_d*` globals → `Q3_FreeMaxBSP` at shutdown. The `#include "surfaceflags.h"` line is commented out, suggesting surface-flag constants are obtained transitively or not needed by consumers of this header.

## External Dependencies
- **`q3files.h`** — defines all `q3_d*_t` struct types, lump constants (`Q3_LUMP_*`), and map size limits (`Q3_MAX_MAP_*`).
- **`struct quakefile_s`** — used by `Q3_LoadBSPFile`; defined elsewhere in the BSPC codebase (not in this header).
- **`byte`**, **`vec3_t`** — primitive typedefs defined elsewhere (likely `qfiles.h` / `q_shared.h`).
- **`surfaceflags.h`** — commented out; surface/content flag bit definitions not directly pulled in here.
