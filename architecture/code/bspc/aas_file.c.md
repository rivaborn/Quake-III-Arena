# code/bspc/aas_file.c

## File Purpose
Handles serialization and deserialization of AAS (Area Awareness System) files for the BSPC tool. It reads and writes the binary AAS navigation mesh format used by the Quake III bot system, including endian-swapping for cross-platform compatibility.

## Core Responsibilities
- Load AAS files from disk into the global `aasworld` structure, lump by lump
- Write AAS data from `aasworld` back to disk in the binary lump format
- Perform little-endian byte-swapping on all AAS data fields
- Apply a lightweight XOR obfuscation pass (`AAS_DData`) to the header on write/read
- Validate file identity (`AASID`) and version (`AASVERSION` / `AASVERSION_OLD`)
- Log reachability counts by travel type and AAS world totals

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `aas_header_t` | struct | AAS file header: ident, version, bspchecksum, lump table |
| `aas_lump_t` | struct | Per-lump offset and length in the file |
| `aas_bbox_t` | struct | Bounding box with presence type and flags |
| `aas_reachability_t` | struct | Reachability link between areas (travel type, start/end, time) |
| `aas_areasettings_t` | struct | Per-area metadata (contents, flags, cluster, reachability range) |
| `aas_portal_t` | struct | Cluster portal area connection |
| `aas_cluster_t` | struct | Cluster grouping of areas |
| `aas_node_t` | struct | BSP tree node for AAS spatial lookup |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `aasworld` | `aas_t` (extern) | global | Central AAS world state; all lumps are stored as fields here; defined in `aas_store` |

## Key Functions / Methods

### AAS_SwapAASData
- **Signature:** `void AAS_SwapAASData(void)`
- **Purpose:** Byte-swaps all integer and float fields in every AAS lump array within `aasworld` for little-endian portability.
- **Inputs:** None (reads/writes `aasworld` directly)
- **Outputs/Return:** None
- **Side effects:** Mutates all numeric fields of every AAS array in `aasworld` in place. Called both on load and before write, making data unusable for rendering after a write pass.
- **Calls:** `LittleLong`, `LittleFloat`, `LittleShort`
- **Notes:** Called twice symmetrically (load → swap to host; write → swap to file). After writing, the in-memory data is in file-endian form and must not be used.

### AAS_DumpAASData
- **Signature:** `void AAS_DumpAASData(void)`
- **Purpose:** Marks the AAS world as unloaded. The actual free calls are commented out, so memory is not released.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Sets `aasworld.loaded = false`. Memory leak risk due to commented-out `FreeMemory` calls.
- **Calls:** None
- **Notes:** Called on error paths to signal unloaded state; does not free pointers.

### AAS_LoadAASLump
- **Signature:** `char *AAS_LoadAASLump(FILE *fp, int offset, int length, void *buf)`
- **Purpose:** Seeks to a lump offset in an open AAS file and reads `length` bytes into `buf`, allocating it if NULL.
- **Inputs:** Open file pointer, byte offset, byte length, optional pre-allocated buffer
- **Outputs/Return:** Pointer to loaded data, or NULL on error
- **Side effects:** May allocate memory via `GetClearedMemory`; calls `AAS_DumpAASData` and `fclose` on error.
- **Calls:** `fseek`, `fread`, `GetClearedMemory`, `FreeMemory`, `AAS_DumpAASData`, `AAS_Error`
- **Notes:** If `length == 0`, returns the existing buffer unchanged with a warning printed.

### AAS_DData
- **Signature:** `void AAS_DData(unsigned char *data, int size)`
- **Purpose:** XOR-obfuscates a byte buffer using `i * 119` as the key per byte.
- **Inputs:** Pointer to data buffer, size in bytes
- **Outputs/Return:** None
- **Side effects:** Mutates `data` in place.
- **Notes:** Applied to the AAS header (bytes 8 onward) on both load and write to obscure lump offsets. Symmetric — applying twice restores original data.

### AAS_LoadAASFile
- **Signature:** `qboolean AAS_LoadAASFile(char *filename, int fpoffset, int fplength)`
- **Purpose:** Opens an AAS file (optionally inside a pak at `fpoffset`), validates header, and loads all 14 lumps into `aasworld`.
- **Inputs:** Path string, pak file offset (0 for standalone), pak file length (unused beyond seek)
- **Outputs/Return:** `qtrue` on success, `qfalse` on any error
- **Side effects:** Allocates memory for all AAS lumps; populates all `aasworld` arrays and counts; sets `aasworld.loaded = true`; calls `AAS_SwapAASData` post-load.
- **Calls:** `AAS_DumpAASData`, `fopen`, `fseek`, `fread`, `LittleLong`, `AAS_DData`, `AAS_LoadAASLump`, `AAS_SwapAASData`, `fclose`, `AAS_Error`
- **Notes:** Supports both `AASVERSION` (5, with header obfuscation) and `AASVERSION_OLD` (4). Reachability, portals, portal index, and clusters may have zero length without failing.

### AAS_WriteAASLump
- **Signature:** `int AAS_WriteAASLump(FILE *fp, aas_header_t *h, int lumpnum, void *data, int length)`
- **Purpose:** Writes one lump's data to the file and records its offset/length in the header's lump table.
- **Inputs:** Open writable file, header pointer, lump index, data buffer, data length in bytes
- **Outputs/Return:** `true` on success, `false` on write error
- **Side effects:** Advances file position; mutates `h->lumps[lumpnum]`; closes file on error.
- **Calls:** `ftell`, `LittleLong`, `fwrite`, `Log_Print`, `fclose`

### AAS_WriteAASFile
- **Signature:** `qboolean AAS_WriteAASFile(char *filename)`
- **Purpose:** Serializes the entire `aasworld` to a binary AAS file, including header obfuscation.
- **Inputs:** Output filename string
- **Outputs/Return:** `qtrue` on success, `qfalse` on failure
- **Side effects:** Calls `AAS_SwapAASData` (corrupting in-memory data for further use); writes file; rewrites header with `AAS_DData` applied; calls `AAS_ShowTotals`.
- **Calls:** `AAS_ShowTotals`, `AAS_SwapAASData`, `memset`, `LittleLong`, `fopen`, `fwrite`, `AAS_WriteAASLump`, `fseek`, `AAS_DData`, `fclose`, `Log_Print`
- **Notes:** After this call, in-memory AAS data is byte-swapped and obfuscated — not safe to use without reloading.

### AAS_ShowNumReachabilities / AAS_ShowTotals
- **Notes:** Diagnostic helpers. `AAS_ShowNumReachabilities` counts reachability links by travel type; `AAS_ShowTotals` logs all AAS lump counts and per-travel-type reachability breakdown via `Log_Print`.

## Control Flow Notes
This file is a **build-time tool** (BSPC compiler), not runtime engine code. It is invoked during map compilation:
- `AAS_LoadAASFile` may be called to reload an existing AAS for incremental processing.
- `AAS_WriteAASFile` is called at the end of AAS generation to persist the result.
- No per-frame or init/shutdown lifecycle — single-pass I/O during map compile.

## External Dependencies
- `qbsp.h` — BSP tool types, `Error`, memory utilities (`GetClearedMemory`, `FreeMemory`), `LittleLong`, `LittleFloat`, `LittleShort`
- `botlib/aasfile.h` — All AAS data structure definitions, lump constants, travel type constants, `AASID`, `AASVERSION`
- `aas_store.h` — Declares `extern aas_t aasworld` (global AAS world state)
- `aas_create.h` — Included for `tmp_aas_t` context; not directly called here
- `Log_Print` — Defined elsewhere (logging subsystem)
- `aasworld` — Defined in `aas_store`; all data arrays populated/read here
