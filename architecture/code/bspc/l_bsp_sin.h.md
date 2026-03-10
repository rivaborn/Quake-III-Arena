# code/bspc/l_bsp_sin.h

## File Purpose
Public interface header for loading, writing, and manipulating Sin (Ritual Entertainment) BSP map files within the BSPC (BSP Compiler) tool. It declares all global BSP lump arrays and the API functions used to process Sin-format `.bsp` files for AAS (Area Awareness System) generation.

## Core Responsibilities
- Declare BSP format magic numbers/version constants for both Sin (`IBSP v41`) and SinGame (`RBSP v1`) variants
- Expose all parsed Sin BSP lump data as `extern` global arrays accessible across translation units
- Declare functions for loading, writing, printing, and vis-compressing Sin BSP data
- Declare entity string parse/unparse utilities for the Sin BSP entity lump

## Key Types / Data Structures

All types are defined in the bundled `sinfiles.h`:

| Name | Kind | Purpose |
|---|---|---|
| `sin_dmodel_t` | struct | Submodel bounding/geometry descriptor |
| `sin_dvis_t` | struct | Visibility lump header (PVS/PHS cluster offsets) |
| `sin_dleaf_t` | struct | BSP leaf node with content/cluster/area info |
| `sin_dplane_t` | struct | BSP split plane (normal + dist + type) |
| `sin_dvertex_t` | struct | Single 3D vertex |
| `sin_dnode_t` | struct | BSP internal node |
| `sin_texinfo_t` | struct | Texture projection + surface properties (SIN-extended) |
| `sin_dface_t` | struct | Face with lightmap styles, edge refs, lightinfo |
| `sin_dedge_t` | struct | Edge defined by two vertex indices |
| `sin_dbrush_t` | struct | Brush (firstside/numsides/contents) |
| `sin_dbrushside_t` | struct | Single brush side (plane + texinfo + lightinfo) |
| `sin_darea_t` | struct | Area with portal list |
| `sin_dareaportal_t` | struct | Portal linking two areas |
| `sin_lightvalue_t` | struct | Per-face light emission/color/style (SIN extension) |
| `sin_lump_t` | struct | Lump offset+length descriptor in BSP header |
| `sin_dheader_t` | struct | Full BSP file header (ident, version, 20 lumps) |

## Global / File-Static State

All declared `extern`; defined in `l_bsp_sin.c`:

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `sin_nummodels` / `sin_dmodels` | `int` / `sin_dmodel_t*` | global | Submodel count and array |
| `sin_visdatasize` / `sin_dvisdata` / `sin_dvis` | `int`/`byte*`/`sin_dvis_t*` | global | Raw vis data + typed vis header pointer |
| `sin_lightdatasize` / `sin_dlightdata` | `int`/`byte*` | global | Lightmap lump |
| `sin_entdatasize` / `sin_dentdata` | `int`/`char*` | global | Entity string lump |
| `sin_numleafs` / `sin_dleafs` | `int`/`sin_dleaf_t*` | global | Leaf lump |
| `sin_numplanes` / `sin_dplanes` | `int`/`sin_dplane_t*` | global | Plane lump |
| `sin_numvertexes` / `sin_dvertexes` | `int`/`sin_dvertex_t*` | global | Vertex lump |
| `sin_numnodes` / `sin_dnodes` | `int`/`sin_dnode_t*` | global | Node lump |
| `sin_numtexinfo` / `sin_texinfo` | `int`/`sin_texinfo_t*` | global | Texinfo lump |
| `sin_numfaces` / `sin_dfaces` | `int`/`sin_dface_t*` | global | Face lump |
| `sin_numedges` / `sin_dedges` | `int`/`sin_dedge_t*` | global | Edge lump |
| `sin_numleaffaces` / `sin_dleaffaces` | `int`/`unsigned short*` | global | Leaf-face index lump |
| `sin_numleafbrushes` / `sin_dleafbrushes` | `int`/`unsigned short*` | global | Leaf-brush index lump |
| `sin_numsurfedges` / `sin_dsurfedges` | `int`/`int*` | global | Surface-edge index lump |
| `sin_numbrushes` / `sin_dbrushes` | `int`/`sin_dbrush_t*` | global | Brush lump |
| `sin_numbrushsides` / `sin_dbrushsides` | `int`/`sin_dbrushside_t*` | global | Brush-side lump |
| `sin_numareas` / `sin_dareas` | `int`/`sin_darea_t*` | global | Area lump |
| `sin_numareaportals` / `sin_dareaportals` | `int`/`sin_dareaportal_t*` | global | Area-portal lump |
| `sin_numlightinfo` / `sin_lightinfo` | `int`/`sin_lightvalue_t*` | global | SIN lightinfo extension lump |
| `sin_dpop` | `byte[256]` | global | POP lump data (legacy Quake2 field) |
| `sin_dbrushsidetextured` | `char[SIN_MAX_MAP_BRUSHSIDES]` | global | Per-brush-side texture presence flags |

## Key Functions / Methods

### Sin_AllocMaxBSP
- Signature: `void Sin_AllocMaxBSP(void)`
- Purpose: Allocates all global BSP lump arrays to their maximum capacities
- Inputs: None
- Outputs/Return: None (side effect only)
- Side effects: Heap allocation for all `sin_d*` arrays
- Calls: Not inferable from this file
- Notes: Must be called before `Sin_LoadBSPFile`

### Sin_FreeMaxBSP
- Signature: `void Sin_FreeMaxBSP(void)`
- Purpose: Frees all heap-allocated BSP lump arrays
- Inputs: None
- Outputs/Return: None
- Side effects: Frees all `sin_d*` allocations
- Calls: Not inferable from this file

### Sin_LoadBSPFile
- Signature: `void Sin_LoadBSPFile(char *filename, int offset, int length)`
- Purpose: Reads and parses a Sin BSP file into the global lump arrays
- Inputs: `filename` — path to `.bsp`; `offset`/`length` — byte range within file (supports embedded BSPs inside PAK)
- Outputs/Return: None; populates all `sin_d*` globals
- Side effects: File I/O; heap allocation via `Sin_AllocMaxBSP`; sets all count/pointer globals
- Calls: Not inferable from this file

### Sin_WriteBSPFile
- Signature: `void Sin_WriteBSPFile(char *filename)`
- Purpose: Serializes all global lump arrays back to a Sin BSP file on disk
- Inputs: `filename` — output path
- Outputs/Return: None
- Side effects: File I/O write
- Calls: Not inferable from this file

### Sin_LoadBSPFileTexinfo
- Signature: `void Sin_LoadBSPFileTexinfo(char *filename)`
- Purpose: Loads only the texinfo lump from a BSP file; comment indicates it is for `qdata` tool use
- Inputs: `filename` — path to `.bsp`
- Outputs/Return: None
- Side effects: Partial file I/O; populates `sin_texinfo`/`sin_numtexinfo`

### Sin_DecompressVis
- Signature: `void Sin_DecompressVis(byte *in, byte *decompressed)`
- Purpose: Run-length decodes a PVS/PHS bitset from the vis lump
- Inputs: `in` — compressed vis data; `decompressed` — output buffer
- Outputs/Return: Decompressed bit vector in `decompressed`
- Side effects: None

### Sin_CompressVis
- Signature: `int Sin_CompressVis(byte *vis, byte *dest)`
- Purpose: Run-length encodes a raw vis bitset for storage in the BSP lump
- Inputs: `vis` — raw bit vector; `dest` — output buffer
- Outputs/Return: Byte length of compressed data
- Side effects: None

### Sin_PrintBSPFileSizes
- Signature: `void Sin_PrintBSPFileSizes(void)`
- Purpose: Prints counts and byte sizes of all lumps to stdout/log for diagnostics
- Inputs: None
- Outputs/Return: None (console I/O)

### Sin_ParseEntities / Sin_UnparseEntities
- Signatures: `void Sin_ParseEntities(void)` / `void Sin_UnparseEntities(void)`
- Purpose: Convert the raw `sin_dentdata` entity string to/from an in-memory entity list structure
- Side effects: Modifies entity list global state (defined elsewhere)

## Control Flow Notes
This header is consumed by the BSPC tool during AAS compilation. Typical flow: `Sin_AllocMaxBSP` → `Sin_LoadBSPFile` (populates all lumps) → BSP-to-AAS conversion routines read the `sin_d*` globals → `Sin_FreeMaxBSP` at shutdown. Not part of the Quake III runtime; strictly a compile-time map-processing tool.

## External Dependencies
- `sinfiles.h` — defines all `sin_d*` struct types, lump index constants (`SIN_LUMP_*`), surface flags (`SURF_*`), `SINHEADER_LUMPS`, and map size limits (`SIN_MAX_MAP_*`)
- `byte`, `vec3_t` — defined elsewhere in shared headers (e.g., `qfiles.h` or `l_utils.h`)
- All function bodies defined in `l_bsp_sin.c` (not present here)
