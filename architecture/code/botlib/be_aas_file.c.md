# code/botlib/be_aas_file.c

## File Purpose
Handles loading and writing of AAS (Area Awareness System) binary navigation files for the Quake III botlib. It reads the lump-based file format into the global `aasworld` structure and performs endian byte-swapping to ensure portability across architectures.

## Core Responsibilities
- Open, validate, and parse AAS files from disk into `aasworld` global state
- Perform little-endian byte-swapping on all loaded AAS data structures
- Free all AAS world data arrays and reset state flags (`AAS_DumpAASData`)
- Write in-memory AAS data back to disk in the lump-based format
- Obfuscate/deobfuscate the file header using a simple XOR cipher (`AAS_DData`)
- Validate AAS file identity, version, and BSP checksum against the loaded map

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `aas_header_t` | struct (extern) | File header: magic, version, BSP checksum, lump directory |
| `aas_lump_t` | struct (extern) | Per-lump offset/length descriptor within the header |
| `aas_bbox_t` | struct (extern) | Bounding box with presence type and flags |
| `aas_plane_t` | struct (extern) | Plane (normal + dist + type) used by AAS geometry |
| `aas_area_t` | struct (extern) | Convex navigation area with bounds and face list |
| `aas_areasettings_t` | struct (extern) | Per-area flags, contents, cluster info, reachability range |
| `aas_reachability_t` | struct (extern) | Directed travel link between two areas |
| `aas_cluster_t` | struct (extern) | Portal-connected group of areas for hierarchical routing |
| `aas_portal_t` | struct (extern) | Boundary area between two clusters |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aasworld` | `aas_world_t` (extern) | global | Central AAS world state; all loaded data arrays live here |
| `AAS_WriteAASLump_offset` | `int` | static (file) | Tracks sequential write offset when writing AAS lumps |

## Key Functions / Methods

### AAS_SwapAASData
- **Signature:** `void AAS_SwapAASData(void)`
- **Purpose:** Byte-swaps every integer and float field in all `aasworld` arrays from/to little-endian order.
- **Inputs:** None (operates on `aasworld` global)
- **Outputs/Return:** void
- **Side effects:** Mutates all data arrays in `aasworld` in-place; data is unusable until swapped back.
- **Calls:** `LittleLong`, `LittleFloat`, `LittleShort`
- **Notes:** Called both on load (convert file → native) and before write (convert native → file). Double-calling is idempotent only on little-endian platforms.

---

### AAS_DumpAASData
- **Signature:** `void AAS_DumpAASData(void)`
- **Purpose:** Frees all dynamically allocated AAS world arrays and resets all count/pointer fields and status flags to zero/false.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Frees hunk memory for every lump array; sets `aasworld.loaded`, `aasworld.initialized`, `aasworld.savefile` to `qfalse`.
- **Calls:** `FreeMemory`
- **Notes:** Called at start of `AAS_LoadAASFile` to flush stale data. Minor bug: `aasworld.numportals` is zeroed twice; `aasworld.portalindexsize` is not zeroed before freeing `portalindex`.

---

### AAS_LoadAASLump
- **Signature:** `char *AAS_LoadAASLump(fileHandle_t fp, int offset, int length, int *lastoffset, int size)`
- **Purpose:** Seeks to a lump's file offset if needed, allocates hunk memory, and reads the raw lump data.
- **Inputs:** `fp` — open file handle; `offset` — lump start in file; `length` — byte count to read; `lastoffset` — in/out sequential position tracker; `size` — element size for dummy alloc when `length==0`.
- **Outputs/Return:** Pointer to allocated buffer, or `NULL` on seek failure.
- **Side effects:** Advances `*lastoffset`; allocates hunk memory; may call `AAS_DumpAASData` and close `fp` on seek error.
- **Calls:** `GetClearedHunkMemory`, `botimport.FS_Seek`, `botimport.FS_Read`, `AAS_Error`, `AAS_DumpAASData`, `botimport.FS_FCloseFile`
- **Notes:** Warns if lumps are not read sequentially. Empty lumps return a minimal dummy allocation rather than NULL.

---

### AAS_DData
- **Signature:** `void AAS_DData(unsigned char *data, int size)`
- **Purpose:** XOR-obfuscates a byte buffer using the formula `data[i] ^= (unsigned char)(i * 119)`.
- **Inputs:** `data` — byte buffer; `size` — buffer length.
- **Outputs/Return:** void
- **Side effects:** Mutates `data` in-place.
- **Notes:** Applied to the AAS header (bytes 8 onward) during both read and write of version `AASVERSION` files. Self-inverse when called twice.

---

### AAS_LoadAASFile
- **Signature:** `int AAS_LoadAASFile(char *filename)`
- **Purpose:** Main entry point — opens an AAS file, validates its header, loads all lumps into `aasworld`, then byte-swaps the data.
- **Inputs:** `filename` — path to the `.aas` file.
- **Outputs/Return:** `BLERR_NOERROR` on success; one of `BLERR_CANNOTOPENAASFILE`, `BLERR_WRONGAASFILEID`, `BLERR_WRONGAASFILEVERSION`, or `BLERR_CANNOTREADAASLUMP` on failure.
- **Side effects:** Populates all `aasworld` lump arrays and counts; sets `aasworld.loaded = qtrue`; conditionally calls `AAS_FileInfo` (debug only).
- **Calls:** `AAS_DumpAASData`, `botimport.FS_FOpenFile`, `botimport.FS_Read`, `LittleLong`, `AAS_DData`, `LibVarGetString`, `atoi`, `AAS_LoadAASLump`, `AAS_SwapAASData`, `botimport.FS_FCloseFile`, `AAS_Error`
- **Notes:** BSP checksum is validated against `sv_mapChecksum` libvar; mismatch returns `BLERR_WRONGAASFILEVERSION` (arguably a misleading error code).

---

### AAS_WriteAASLump
- **Signature:** `int AAS_WriteAASLump(fileHandle_t fp, aas_header_t *h, int lumpnum, void *data, int length)`
- **Purpose:** Records a lump's offset/length into the header struct and writes its raw bytes to the file.
- **Inputs:** `fp` — open write handle; `h` — header to update in memory; `lumpnum` — `AASLUMP_*` index; `data` — bytes to write; `length` — byte count.
- **Outputs/Return:** `qtrue` always.
- **Side effects:** Mutates `h->lumps[lumpnum]`; advances `AAS_WriteAASLump_offset`; writes to `fp`.
- **Calls:** `botimport.FS_Write`, `LittleLong`

---

### AAS_WriteAASFile
- **Signature:** `qboolean AAS_WriteAASFile(char *filename)`
- **Purpose:** Serializes the current `aasworld` data to an AAS file on disk, writing a placeholder header first, then all lumps, then rewriting the completed header.
- **Inputs:** `filename` — output file path.
- **Outputs/Return:** `qtrue` on success, `qfalse` if file cannot be opened.
- **Side effects:** Calls `AAS_SwapAASData` (data is byte-swapped and unusable after); writes and closes file; obfuscates header with `AAS_DData`.
- **Calls:** `AAS_SwapAASData`, `Com_Memset`, `LittleLong`, `botimport.FS_FOpenFile`, `botimport.FS_Write`, `AAS_WriteAASLump`, `botimport.FS_Seek`, `AAS_DData`, `botimport.FS_FCloseFile`
- **Notes:** Comment explicitly warns that AAS data is corrupted (byte-swapped) after this call and must not be used further.

## Control Flow Notes
This file is used during **initialization**: `AAS_LoadAASFile` is called once when a map loads to populate the navigation world. `AAS_DumpAASData` is also called during shutdown or map change. `AAS_WriteAASFile` is a tool/offline-use path (called from the AAS optimizer/cluster builder, not during normal gameplay). The file has no per-frame involvement.

## External Dependencies
- `aasfile.h` — `AASID`, `AASVERSION`, `AASVERSION_OLD`, `AASLUMP_*` constants, `aas_header_t`, lump type definitions
- `be_aas_def.h` — defines `aasworld` global of type `aas_world_t`
- `be_interface.h` — `botimport` (FS I/O, print); `AAS_Error`
- `be_aas_funcs.h` — `AAS_Error` declaration
- `l_libvar.h` — `LibVarGetString` (reads `sv_mapChecksum`)
- `l_memory.h` — `GetClearedHunkMemory`, `FreeMemory`
- `q_shared.h` — `LittleLong`, `LittleFloat`, `LittleShort`, `fileHandle_t`, `qboolean`, `FS_READ/WRITE/SEEK_SET`, `Com_Memset`
- `botimport` (defined elsewhere) — virtual filesystem and print callbacks used throughout
