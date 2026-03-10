# code/bspc/l_bsp_q2.h

## File Purpose
Public header for the Quake II BSP file interface used by the BSPC (BSP Compiler) tool. It declares all globally shared BSP lump arrays and their associated count variables, along with the functions needed to load, write, and manipulate Q2 BSP data.

## Core Responsibilities
- Declare all extern BSP lump data arrays (geometry, visibility, lighting, entities, etc.)
- Expose Q2 BSP file I/O functions (`Load`, `Write`, `Print`)
- Expose visibility compression/decompression routines
- Expose entity string parse/unparse utilities
- Provide memory management entry points (`AllocMaxBSP`, `FreeMaxBSP`)

## Key Types / Data Structures
None defined here; all types are defined elsewhere (`dmodel_t`, `dleaf_t`, `dplane_t`, `dnode_t`, `dface_t`, `dedge_t`, `dbrush_t`, `dbrushside_t`, `darea_t`, `dareaportal_t`, `dvertex_t`, `texinfo_t`, `dvis_t`, `dareaportal_t`).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `nummodels` / `dmodels` | `int` / `dmodel_t *` | global | Sub-model count and array |
| `visdatasize` / `dvisdata` / `dvis` | `int` / `byte *` / `dvis_t *` | global | Raw and structured visibility data |
| `lightdatasize` / `dlightdata` | `int` / `byte *` | global | Lightmap blob |
| `entdatasize` / `dentdata` | `int` / `char *` | global | Entity string lump |
| `numleafs` / `dleafs` | `int` / `dleaf_t *` | global | BSP leaf nodes |
| `numplanes` / `dplanes` | `int` / `dplane_t *` | global | Splitting planes |
| `numvertexes` / `dvertexes` | `int` / `dvertex_t *` | global | Vertex positions |
| `numnodes` / `dnodes` | `int` / `dnode_t *` | global | Interior BSP nodes |
| `numtexinfo` / `texinfo` | `int` / `texinfo_t[]` | global | Texture mapping info (static array, not pointer) |
| `numfaces` / `dfaces` | `int` / `dface_t *` | global | Polygon faces |
| `numedges` / `dedges` | `int` / `dedge_t *` | global | Edge list |
| `numleaffaces` / `dleaffaces` | `int` / `unsigned short *` | global | Leaf-to-face index list |
| `numleafbrushes` / `dleafbrushes` | `int` / `unsigned short *` | global | Leaf-to-brush index list |
| `numsurfedges` / `dsurfedges` | `int` / `int *` | global | Surface edge references |
| `numareas` / `dareas` | `int` / `darea_t *` | global | PVS areas |
| `numareaportals` / `dareaportals` | `int` / `dareaportal_t *` | global | Inter-area portals |
| `numbrushes` / `dbrushes` | `int` / `dbrush_t *` | global | Convex brush list |
| `numbrushsides` / `dbrushsides` | `int` / `dbrushside_t *` | global | Brush side planes |
| `dpop[256]` | `byte[256]` | global | Vis decompression lookup table |
| `brushsidetextured` | `char[MAX_MAP_BRUSHSIDES]` | global | Per-brush-side texture presence flags |

## Key Functions / Methods

### Q2_AllocMaxBSP / Q2_FreeMaxBSP
- Signature: `void Q2_AllocMaxBSP(void)` / `void Q2_FreeMaxBSP(void)`
- Purpose: Allocate and free the maximum-sized BSP lump buffers.
- Inputs: None
- Outputs/Return: None
- Side effects: Allocates/frees all global lump pointer arrays.
- Calls: Not inferable from this file.
- Notes: Must be called before any BSP load or write operation.

### Q2_LoadBSPFile
- Signature: `void Q2_LoadBSPFile(char *filename, int offset, int length)`
- Purpose: Read a Q2 BSP file from disk into the global lump arrays.
- Inputs: File path, optional byte offset and length (for embedded BSPs).
- Outputs/Return: None; populates all global `d*` arrays and count variables.
- Side effects: Disk I/O; fills global state.
- Calls: Not inferable from this file.

### Q2_WriteBSPFile
- Signature: `void Q2_WriteBSPFile(char *filename)`
- Purpose: Serialize all global lump arrays back to a Q2 BSP file on disk.
- Inputs: Output file path.
- Outputs/Return: None.
- Side effects: Disk I/O.
- Calls: Not inferable from this file.

### Q2_DecompressVis / Q2_CompressVis
- Signature: `void Q2_DecompressVis(byte *in, byte *decompressed)` / `int Q2_CompressVis(byte *vis, byte *dest)`
- Purpose: Run-length decode/encode the PVS (Potentially Visible Set) bitfield for a BSP leaf.
- Inputs: Compressed or raw vis buffer pointers.
- Outputs/Return: `Q2_CompressVis` returns compressed byte count; `Q2_DecompressVis` writes to output buffer.
- Side effects: None beyond output buffer writes.

### Q2_ParseEntities / Q2_UnparseEntities
- Signature: `void Q2_ParseEntities(void)` / `void Q2_UnparseEntities(void)`
- Purpose: Convert the raw `dentdata` entity string lump to/from an internal entity list representation.
- Side effects: Modifies global entity list state and/or `dentdata`.

### Q2_LoadBSPFileTexinfo / Q2_PrintBSPFileSizes
- Notes: `Q2_LoadBSPFileTexinfo` loads only the texinfo lump (documented as "just for qdata"). `Q2_PrintBSPFileSizes` dumps lump sizes to console/log for diagnostics.

## Control Flow Notes
This header is consumed during BSPC tool initialization. `Q2_AllocMaxBSP` is called first, then `Q2_LoadBSPFile` populates all lumps. The BSPC compiler then reads the global arrays to build its AAS navigation data. `Q2_WriteBSPFile` may be called to output a modified BSP. This file is not part of the runtime game engine; it belongs to the offline build toolchain.

## External Dependencies
- BSP lump types (`dmodel_t`, `dleaf_t`, `dplane_t`, `dnode_t`, `dface_t`, `dedge_t`, `dbrush_t`, `dbrushside_t`, `darea_t`, `dareaportal_t`, `dvis_t`, `texinfo_t`, `dvertex_t`, `dedge_t`) — defined elsewhere, likely `aasfile.h` / `q3files.h` / Q2 BSP format headers
- `MAX_MAP_*` constants — defined elsewhere (BSP format limits header)
- `byte` typedef — defined elsewhere (likely `q_shared.h`)
