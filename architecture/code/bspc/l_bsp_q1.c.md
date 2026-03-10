# code/bspc/l_bsp_q1.c

## File Purpose
Implements loading, saving, and manipulation of Quake 1 BSP (Binary Space Partition) map files within the BSPC tool. It handles memory allocation for all BSP lumps, byte-swapping between little-endian disk format and host byte order, and serialization/deserialization of entity data.

## Core Responsibilities
- Allocate and free max-capacity BSP data arrays for all Q1 lump types
- Load a Q1 BSP file from disk, copying each lump into pre-allocated global arrays
- Write a Q1 BSP file to disk, serializing all lump arrays with proper byte ordering
- Byte-swap all BSP lump data between disk (little-endian) and host formats
- Parse the entity string lump into an in-memory `entities[]` array
- Serialize the in-memory entity array back into the entity string lump

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `q1_dmodel_t` | struct (typedef, defined in `l_bsp_q1.h`) | Per-model BSP record (headnodes, bounds, face range) |
| `q1_dheader_t` | struct (typedef) | BSP file header containing lump directory entries |
| `q1_lump_t` | struct (typedef) | Single lump descriptor: file offset + byte length |
| `q1_dmiptexlump_t` | struct (typedef) | Miptex lump header: count + per-texture offsets |
| `q1_dleaf_t` | struct (typedef) | BSP leaf: contents, bounds, marksurface range, vis offset |
| `q1_dplane_t` | struct (typedef) | Plane: normal, distance, type |
| `q1_dnode_t` | struct (typedef) | BSP interior node: plane, children, bounds, face range |
| `q1_dclipnode_t` | struct (typedef) | Clip hull node: plane + two children |
| `q1_dface_t` | struct (typedef) | Polygon face: plane, texinfo, edges, light offset |
| `q1_dedge_t` | struct (typedef) | Edge: two vertex indices |
| `q1_texinfo_t` | struct (typedef) | Texture mapping vectors, miptex index, flags |
| `q1_dvertex_t` | struct (typedef) | 3D vertex position |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `q1_nummodels` / `q1_dmodels` | `int` / `q1_dmodel_t *` | global | Model lump count and array |
| `q1_visdatasize` / `q1_dvisdata` | `int` / `byte *` | global | Visibility lump byte count and data |
| `q1_lightdatasize` / `q1_dlightdata` | `int` / `byte *` | global | Lightmap lump byte count and data |
| `q1_texdatasize` / `q1_dtexdata` | `int` / `byte *` | global | Miptex lump byte count and data |
| `q1_entdatasize` / `q1_dentdata` | `int` / `char *` | global | Entity string byte count and buffer |
| `q1_numleafs` … `q1_numsurfedges` | `int` (×9) | global | Per-lump element counts |
| `q1_dleafs` … `q1_dsurfedges` | typed pointers (×9) | global | Per-lump data arrays |
| `q1_bspallocated` | `int` | global | Allocation state flag (unused after set) |
| `q1_allocatedbspmem` | `int` | global | Running total of allocated BSP bytes |
| `q1_header` | `q1_dheader_t *` | global (file-level) | Pointer to loaded/output BSP header |
| `q1_fileLength` | `int` | global (file-level) | Byte length of loaded BSP file |
| `q1_wadfile` | `FILE *` | global (file-level) | Output file handle for BSP writing |
| `q1_outheader` | `q1_dheader_t` | global (file-level) | Static output header buffer |

## Key Functions / Methods

### Q1_AllocMaxBSP
- **Signature:** `void Q1_AllocMaxBSP(void)`
- **Purpose:** Allocates maximum-capacity heap buffers for every Q1 BSP lump array.
- **Inputs:** None.
- **Outputs/Return:** `void`; global lump pointers and counts initialized.
- **Side effects:** Calls `GetMemory` for each lump; accumulates total into `q1_allocatedbspmem`; logs allocation size. Note: there is a bug on the edges accumulation line (`q1_allocatedbspmem += Q1_MAX_MAP_EDGES, sizeof(q1_dedge_t)` uses comma operator instead of `*`).
- **Calls:** `GetMemory`, `Log_Print`, `PrintMemorySize`
- **Notes:** Must be called before `Q1_LoadBSPFile`.

### Q1_FreeMaxBSP
- **Signature:** `void Q1_FreeMaxBSP(void)`
- **Purpose:** Frees all BSP lump buffers and resets all counts to zero.
- **Inputs:** None.
- **Outputs/Return:** `void`
- **Side effects:** Calls `FreeMemory` for each pointer; zeroes `q1_allocatedbspmem`; logs freed size.
- **Calls:** `FreeMemory`, `Log_Print`, `PrintMemorySize`

### Q1_SwapBSPFile
- **Signature:** `void Q1_SwapBSPFile(qboolean todisk)`
- **Purpose:** Byte-swaps all in-memory BSP lump data between host byte order and little-endian disk format. Called after load (todisk=false) and before write (todisk=true).
- **Inputs:** `todisk` — direction of swap; affects miptex count reading order.
- **Outputs/Return:** `void`; all global lump arrays modified in-place.
- **Side effects:** Modifies all global lump arrays directly.
- **Calls:** `LittleLong`, `LittleFloat`, `LittleShort`
- **Notes:** Miptex swap uses `todisk` to correctly handle the `nummiptex` field before or after swapping it.

### Q1_CopyLump
- **Signature:** `int Q1_CopyLump(int lump, void *dest, int size, int maxsize)`
- **Purpose:** Copies a single lump from the loaded BSP file image into a destination buffer, with bounds checking.
- **Inputs:** `lump` — lump index; `dest` — target buffer; `size` — per-element byte size; `maxsize` — max element count.
- **Outputs/Return:** Number of elements copied.
- **Side effects:** `memcpy` into `dest`; prints warnings if limits exceeded; calls `Error` if lump size is misaligned.
- **Calls:** `memcpy`, `printf`, `Error`

### Q1_LoadBSPFile
- **Signature:** `void Q1_LoadBSPFile(char *filename, int offset, int length)`
- **Purpose:** Loads a Q1 BSP file from disk into global arrays, validating the header version and byte-swapping all data.
- **Inputs:** `filename` — path; `offset`/`length` — subrange within file (for embedded BSPs).
- **Outputs/Return:** `void`; all global lump arrays populated.
- **Side effects:** Allocates file buffer via `LoadFile`; frees it after copying; calls `Q1_SwapBSPFile`; calls `Error` on version mismatch.
- **Calls:** `LoadFile`, `Q1_CopyLump`, `FreeMemory`, `Q1_SwapBSPFile`, `Error`, `LittleLong`

### Q1_WriteBSPFile
- **Signature:** `void Q1_WriteBSPFile(char *filename)`
- **Purpose:** Writes all global BSP lump arrays to a Q1 BSP file, updating the header lump directory at the end.
- **Inputs:** `filename` — output path.
- **Outputs/Return:** `void`
- **Side effects:** Swaps data to disk order via `Q1_SwapBSPFile(true)`; opens/writes/closes `q1_wadfile`; final `fseek`+write overwrites the header.
- **Calls:** `Q1_SwapBSPFile`, `SafeOpenWrite`, `SafeWrite`, `Q1_AddLump`, `fseek`, `fclose`
- **Notes:** Data is swapped in-place and should not be used after this call without re-loading.

### Q1_AddLump
- **Signature:** `void Q1_AddLump(int lumpnum, void *data, int len)`
- **Purpose:** Appends a single lump's data to the open output file and records its offset/length in the header.
- **Inputs:** `lumpnum` — lump slot index; `data` — buffer; `len` — byte count.
- **Side effects:** `ftell`/`SafeWrite` on `q1_wadfile`; updates `q1_header->lumps[lumpnum]`.
- **Calls:** `ftell`, `LittleLong`, `SafeWrite`

### Q1_ParseEntities
- **Signature:** `void Q1_ParseEntities(void)`
- **Purpose:** Parses `q1_dentdata` entity string into the global `entities[]` array using the script lexer.
- **Side effects:** Resets `num_entities`; populates `entities[]` (defined in `l_bsp_ent`).
- **Calls:** `LoadScriptMemory`, `SetScriptFlags`, `ParseEntity`, `FreeScript`

### Q1_UnparseEntities
- **Signature:** `void Q1_UnparseEntities(void)`
- **Purpose:** Serializes the in-memory `entities[]` array back into the `q1_dentdata` string buffer.
- **Side effects:** Overwrites `q1_dentdata`; updates `q1_entdatasize`; calls `Error` on overflow.
- **Calls:** `strcat`, `sprintf`, `strlen`, `Error`
- **Notes:** Uses unsafe `strcat` with manual pointer advancement; buffer overflow risk if entity data exceeds `Q1_MAX_MAP_ENTSTRING`.

## Control Flow Notes
This file is used exclusively by the BSPC offline tool (not the game runtime). The typical flow is:
1. `Q1_AllocMaxBSP()` — called once at startup to reserve buffers.
2. `Q1_LoadBSPFile()` — loads and byte-swaps a `.bsp` file into globals.
3. Optionally `Q1_ParseEntities()` / `Q1_UnparseEntities()` for entity manipulation.
4. `Q1_WriteBSPFile()` — writes modified data back out.
5. `Q1_FreeMaxBSP()` — releases all buffers at shutdown.

## External Dependencies
- `l_cmd.h` — `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `Error`, `LittleLong`, `LittleFloat`, `LittleShort`, `qboolean`, `byte`
- `l_mem.h` — `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `botlib/l_script.h` — `script_t`, `LoadScriptMemory`, `SetScriptFlags`, `FreeScript`
- `l_bsp_q1.h` — all `q1_d*` type definitions, lump index constants, version constant (defined elsewhere)
- `l_bsp_ent.h` — `entities[]`, `num_entities`, `epair_t`, `ParseEntity` (defined elsewhere)
