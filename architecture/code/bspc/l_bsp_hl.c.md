# code/bspc/l_bsp_hl.c

## File Purpose
Implements loading, saving, and manipulation of Half-Life (GoldSrc) BSP format files for use in the BSPC (BSP compiler/converter) tool. It handles all lump types defined in the HL BSP format and bridges them into the BSPC entity parsing system.

## Core Responsibilities
- Allocate and free max-capacity buffers for all HL BSP lump types
- Load a Half-Life BSP file from disk into global arrays, with bounds checking
- Write a Half-Life BSP file from global arrays back to disk
- Perform little-endian byte swapping on all structured BSP lumps
- Compress and decompress visibility data (run-length encoding)
- Compute fast XOR-shift checksums for each loaded lump
- Parse and unparse the entity string lump using the botlib script system

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `hl_dmodel_t` | struct (typedef, defined in l_bsp_hl.h) | Per-model header: headnodes, bounds, face range |
| `hl_dleaf_t` | struct | BSP leaf: contents, bounds, vis offset, mark surface range |
| `hl_dnode_t` | struct | BSP node: plane, children, bounds, face range |
| `hl_dplane_t` | struct | Plane: normal, dist, type |
| `hl_dface_t` | struct | Polygon face: plane, edges, texinfo, lightmap offset |
| `hl_dclipnode_t` | struct | Clip tree node: plane, children (used for hull collision) |
| `hl_dedge_t` | struct | Edge: two vertex indices |
| `hl_texinfo_t` | struct | Texture projection vectors, miptex index, flags |
| `hl_dmiptexlump_t` | struct | Header for miptex data block: count + offsets |
| `hl_dheader_t` | struct | BSP file header: version + lump directory |
| `hl_lump_t` | struct | Lump descriptor: file offset + length |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `hl_nummodels` / `hl_dmodels` | `int` / `hl_dmodel_t *` | global | Model lump count and data |
| `hl_numleafs` / `hl_dleafs` | `int` / `hl_dleaf_t *` | global | Leaf lump |
| `hl_numnodes` / `hl_dnodes` | `int` / `hl_dnode_t *` | global | Node lump |
| `hl_numplanes` / `hl_dplanes` | `int` / `hl_dplane_t *` | global | Plane lump |
| `hl_numfaces` / `hl_dfaces` | `int` / `hl_dface_t *` | global | Face lump |
| `hl_numclipnodes` / `hl_dclipnodes` | `int` / `hl_dclipnode_t *` | global | Clip node lump |
| `hl_numedges` / `hl_dedges` | `int` / `hl_dedge_t *` | global | Edge lump |
| `hl_nummarksurfaces` / `hl_dmarksurfaces` | `int` / `unsigned short *` | global | Mark-surface lump |
| `hl_numsurfedges` / `hl_dsurfedges` | `int` / `int *` | global | Surface-edge lump |
| `hl_numtexinfo` / `hl_texinfo` | `int` / `hl_texinfo_t *` | global | Texinfo lump |
| `hl_visdatasize` / `hl_dvisdata` | `int` / `byte *` | global | Visibility lump (raw bytes) |
| `hl_lightdatasize` / `hl_dlightdata` | `int` / `byte *` | global | Lightmap lump |
| `hl_texdatasize` / `hl_dtexdata` | `int` / `byte *` | global | Miptex lump |
| `hl_entdatasize` / `hl_dentdata` | `int` / `char *` | global | Entity string lump |
| `hl_bspallocated` / `hl_allocatedbspmem` | `int` | global | Allocation state flags and byte total |
| `hl_header` | `hl_dheader_t *` | file-static global | Pointer to in-memory BSP header during load/write |
| `hl_fileLength` | `int` | file-static global | Total byte length of loaded BSP file |
| `wadfile` | `FILE *` | file-static global | Output file handle during `HL_WriteBSPFile` |
| `outheader` | `hl_dheader_t` | file-static global | Output header written at start of BSP write |
| Per-lump `*_checksum` variables | `int` | global | XOR-shift checksum for each lump after load |

## Key Functions / Methods

### HL_AllocMaxBSP
- **Signature:** `void HL_AllocMaxBSP(void)`
- **Purpose:** Allocates worst-case heap buffers for every HL BSP lump type.
- **Inputs:** None.
- **Outputs/Return:** None. Populates all global `hl_d*` pointers.
- **Side effects:** Calls `GetMemory` (custom allocator) for each lump; logs total allocated size via `Log_Print`/`PrintMemorySize`.
- **Calls:** `GetMemory`, `Log_Print`, `PrintMemorySize`
- **Notes:** Bug present — edge allocation uses comma operator instead of `+=`: `hl_allocatedbspmem += HL_MAX_MAP_EDGES, sizeof(hl_dedge_t);` (intended `*`).

### HL_FreeMaxBSP
- **Signature:** `void HL_FreeMaxBSP(void)`
- **Purpose:** Frees all lump buffers allocated by `HL_AllocMaxBSP`, resets counts/pointers to zero/NULL.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Calls `FreeMemory` for every buffer; logs freed size; resets `hl_allocatedbspmem` to 0.
- **Calls:** `FreeMemory`, `Log_Print`, `PrintMemorySize`

### FastChecksum
- **Signature:** `int FastChecksum(void *buffer, int bytes)`
- **Purpose:** Computes a simple XOR-shift checksum over a byte buffer.
- **Inputs:** `buffer` — data to checksum; `bytes` — byte count.
- **Outputs/Return:** 32-bit checksum integer.
- **Side effects:** None.
- **Notes:** Not cryptographically sound; used for quick change detection between BSP loads.

### HL_CompressVis
- **Signature:** `int HL_CompressVis(byte *vis, byte *dest)`
- **Purpose:** Run-length encodes a PVS row (zero-runs only) per HL vis format.
- **Inputs:** `vis` — uncompressed row (`visrow` bytes); `dest` — output buffer.
- **Outputs/Return:** Byte length of compressed data.
- **Side effects:** Reads `hl_numleafs` global.

### HL_DecompressVis
- **Signature:** `void HL_DecompressVis(byte *in, byte *decompressed)`
- **Purpose:** Expands a run-length encoded PVS row back to raw bytes.
- **Inputs:** `in` — compressed data; `decompressed` — output buffer (`row` bytes).
- **Side effects:** Reads `hl_numleafs` global.

### HL_SwapBSPFile
- **Signature:** `void HL_SwapBSPFile(qboolean todisk)`
- **Purpose:** Byte-swaps all structured BSP lump data in-place for cross-endian I/O.
- **Inputs:** `todisk` — if true, native→little-endian; if false, little-endian→native.
- **Outputs/Return:** None.
- **Side effects:** Mutates all global `hl_d*` arrays in-place.
- **Calls:** `LittleLong`, `LittleFloat`, `LittleShort`
- **Notes:** Miptex header swap uses `todisk` to decide whether `nummiptex` is already swapped before iterating offsets.

### HL_CopyLump
- **Signature:** `int HL_CopyLump(int lump, void *dest, int size, int maxsize)`
- **Purpose:** Copies one lump from the memory-mapped BSP header into a destination buffer, with overflow guards.
- **Inputs:** `lump` — lump index; `dest` — destination; `size` — element size; `maxsize` — element capacity.
- **Outputs/Return:** Number of elements copied.
- **Side effects:** Prints warnings on overflow; calls `Error` on unaligned lump sizes; reads `hl_header` and `hl_fileLength` globals.
- **Calls:** `memcpy`, `Error`, `printf`

### HL_LoadBSPFile
- **Signature:** `void HL_LoadBSPFile(char *filename, int offset, int length)`
- **Purpose:** Loads a Half-Life BSP file from disk, copies all lumps into global arrays, byte-swaps, and computes per-lump checksums.
- **Inputs:** `filename` — path; `offset`, `length` — byte range within file (0/0 = whole file).
- **Outputs/Return:** None.
- **Side effects:** Allocates file buffer via `LoadFile`; frees it via `FreeMemory` after copy; populates all `hl_*` globals; calls `HL_SwapBSPFile(false)`.
- **Calls:** `LoadFile`, `LittleLong`, `Error`, `HL_CopyLump`, `FreeMemory`, `HL_SwapBSPFile`, `FastChecksum`
- **Notes:** Bug — `hl_dtexdata_checksum` uses `hl_numedges` instead of `hl_texdatasize` as byte count.

### HL_WriteBSPFile
- **Signature:** `void HL_WriteBSPFile(char *filename)`
- **Purpose:** Serializes all global BSP data to a Half-Life BSP file on disk.
- **Inputs:** `filename` — output path.
- **Outputs/Return:** None.
- **Side effects:** Calls `HL_SwapBSPFile(true)`; writes header placeholder, all lumps via `HL_AddLump`, then seeks back and overwrites header; closes file.
- **Calls:** `HL_SwapBSPFile`, `SafeOpenWrite`, `SafeWrite`, `HL_AddLump`, `fseek`, `fclose`

### HL_AddLump
- **Signature:** `void HL_AddLump(int lumpnum, void *data, int len)`
- **Purpose:** Appends one lump to the open WAD/BSP output file and records its offset/length in the header.
- **Inputs:** `lumpnum`, `data`, `len`.
- **Side effects:** Writes data padded to 4-byte alignment; updates `hl_header->lumps[lumpnum]`.
- **Calls:** `ftell`, `LittleLong`, `SafeWrite`

### HL_PrintBSPFileSizes
- **Signature:** `void HL_PrintBSPFileSizes(void)`
- **Purpose:** Prints a formatted usage table for all lump types to stdout.
- **Calls:** `ArrayUsage`, `GlobUsage`, `qprintf`

### HL_ParseEntities
- **Signature:** `void HL_ParseEntities(void)`
- **Purpose:** Parses the `hl_dentdata` entity string into the shared `entities[]` array using the botlib script system.
- **Calls:** `LoadScriptMemory`, `SetScriptFlags`, `ParseEntity`, `FreeScript`

### HL_UnparseEntities
- **Signature:** `void HL_UnparseEntities(void)`
- **Purpose:** Serializes `entities[]` back into `hl_dentdata` in Quake key-value text format.
- **Side effects:** Overwrites `hl_dentdata`; updates `hl_entdatasize`.
- **Calls:** `strcat`, `sprintf`, `strlen`, `Error`

## Control Flow Notes
This file is a **tool-time** (offline) module used only by the BSPC compiler, not the game runtime. It is invoked during map conversion: `HL_AllocMaxBSP` → `HL_LoadBSPFile` → (conversion logic) → `HL_WriteBSPFile` → `HL_FreeMaxBSP`. There is no frame/update loop involvement.

## External Dependencies
- `l_cmd.h` — `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `Error`, `LittleLong`, `LittleShort`, `LittleFloat`, `qprintf`
- `l_math.h` — math types (`vec3_t`, etc.; not directly used in this file)
- `l_mem.h` — `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `botlib/l_script.h` — `LoadScriptMemory`, `SetScriptFlags`, `FreeScript`
- `l_bsp_hl.h` — all `hl_d*` struct type definitions and lump index constants (defined elsewhere)
- `l_bsp_ent.h` — `ParseEntity`, `entities[]`, `num_entities`, `epair_t` (defined elsewhere)
