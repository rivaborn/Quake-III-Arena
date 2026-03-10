# code/bspc/textures.c

## File Purpose
Provides texture resolution and texinfo generation utilities for the BSPC (BSP Compiler) tool. It maps brush surface texture names to miptex metadata and computes axis-aligned texture projection vectors used when writing BSP lumps.

## Core Responsibilities
- Cache and deduplicate loaded miptex entries (name, flags, value, contents, animation chain)
- Load `.wal` texture files from disk to extract surface flags/contents metadata
- Compute texture projection axes from a plane normal using a best-fit axis table
- Build and deduplicate `texinfo_t` records from brush texture parameters (scale, rotate, shift, origin offset)
- Recursively resolve animated texture chains via `nexttexinfo` linkage

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `textureref_t` | typedef struct (defined in `qbsp.h`) | Caches per-texture name, flags, value, contents, and animation name loaded from `.wal` |
| `brush_texture_t` | typedef struct (defined in `qbsp.h`) | Input surface parameters: shift, rotate, scale, name, flags, value |
| `plane_t` | typedef struct | BSP plane with normal, dist, type, signbits |
| `texinfo_t` | struct (from `q2files.h` via `l_bsp_q2.h`) | Output texinfo record: projection vectors, flags, value, texture name, animation chain |
| `miptex_t` | struct (from `q2files.h`) | On-disk `.wal` texture header with flags, value, contents, animname |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `nummiptex` | `int` | global | Count of loaded/registered miptex entries |
| `textureref[MAX_MAP_TEXTURES]` | `textureref_t[1024]` | global | Flat cache of all textures encountered during map compilation |
| `baseaxis[18]` | `vec3_t[18]` | global (file-level) | 6 axis-triple sets used for planar texture projection axis selection |

## Key Functions / Methods

### FindMiptex
- **Signature:** `int FindMiptex(char *name)`
- **Purpose:** Looks up or registers a texture by name in `textureref[]`, loading its `.wal` metadata from disk if not already cached.
- **Inputs:** `name` — texture name string (no path/extension)
- **Outputs/Return:** Index into `textureref[]`
- **Side effects:** Increments `nummiptex`; allocates/frees a `miptex_t` via `TryLoadFile`/`FreeMemory`; may recurse to register the animation target
- **Calls:** `TryLoadFile`, `LittleLong`, `FreeMemory`, `FindMiptex` (recursive for `animname`), `Error`
- **Notes:** Aborts with `Error` if `MAX_MAP_TEXTURES` (1024) is exceeded. Silently skips metadata if `.wal` file is absent.

### TextureAxisFromPlane
- **Signature:** `void TextureAxisFromPlane(plane_t *pln, vec3_t xv, vec3_t yv)`
- **Purpose:** Selects the best-fit texture projection axes for a plane by comparing the plane normal against six cardinal axis normals in `baseaxis`.
- **Inputs:** `pln` — plane with normal; `xv`, `yv` — output vectors
- **Outputs/Return:** Writes best S and T axis vectors into `xv` and `yv`
- **Side effects:** None
- **Calls:** `DotProduct`, `VectorCopy`
- **Notes:** Standard Quake/Quake2 planar projection axis selection; operates on `baseaxis` which encodes floor, ceiling, and four wall orientations.

### TexinfoForBrushTexture
- **Signature:** `int TexinfoForBrushTexture(plane_t *plane, brush_texture_t *bt, vec3_t origin)`
- **Purpose:** Computes a fully transformed `texinfo_t` from a brush face's texture parameters (shift, rotate, scale, origin offset) and deduplicates it against the global `texinfo[]` array.
- **Inputs:** `plane` — face plane; `bt` — brush texture params; `origin` — entity origin offset
- **Outputs/Return:** Index into the global `texinfo[]` array (defined in `l_bsp_q2.h`)
- **Side effects:** May append to global `texinfo[]` and increment `numtexinfo`; recursively calls itself to register the animation chain's `nexttexinfo` entry
- **Calls:** `TextureAxisFromPlane`, `DotProduct`, `sin`, `cos`, `FindMiptex`, `TexinfoForBrushTexture` (recursive), `memset`, `strcmp`
- **Notes:** Returns 0 immediately for empty texture names. Rotation is computed with trig for non-cardinal angles. Uses `goto skip` for inner-loop early-exit during deduplication search.

## Control Flow Notes
Used during map-to-BSP compilation in the BSPC tool. Called from brush/side processing code (e.g., `map_q2.c`, `map_q1.c`) when constructing `side_t` texinfo references. Not part of the runtime game engine; executes entirely at BSP build time, before tree construction.

## External Dependencies
- `qbsp.h` — `plane_t`, `brush_texture_t`, `textureref_t`, `MAX_MAP_TEXTURES`, common BSPC types and declarations
- `l_bsp_q2.h` — `texinfo_t`, `numtexinfo`, global BSP lump arrays (`texinfo[]`)
- `q2files.h` (via `qbsp.h`) — `miptex_t` on-disk layout, `texinfo_t` definition
- `TryLoadFile`, `FreeMemory` — defined in memory/file utility modules (`l_mem`, `l_qfiles`)
- `Error` — fatal error handler, defined elsewhere
- `gamedir` — global string for game data path, defined elsewhere
- `DotProduct`, `VectorCopy` — math macros/functions from `l_math.h`
- `LittleLong` — endian swap macro, defined elsewhere
