# code/bspc/l_bsp_sin.c

## File Purpose
Implements BSP file I/O and in-memory storage for the SiN game engine BSP format within the BSPC tool. It handles loading, writing, byte-swapping, and memory management for all SiN BSP lumps, and includes geometry helpers for texture reference fixing.

## Core Responsibilities
- Allocate and free max-capacity BSP data arrays (`Sin_AllocMaxBSP` / `Sin_FreeMaxBSP`)
- Load a SiN BSP file from disk into global arrays (`Sin_LoadBSPFile`)
- Write global BSP arrays back to disk (`Sin_WriteBSPFile`)
- Byte-swap all BSP data between little-endian disk format and host format (`Sin_SwapBSPFile`)
- Compress and decompress PVS visibility data (`Sin_CompressVis` / `Sin_DecompressVis`)
- Parse and unparse entity key-value pairs from the entity lump string (`Sin_ParseEntities` / `Sin_UnparseEntities`)
- Fix brush-side texture references by matching faces to brush sides geometrically (`Sin_FixTextureReferences`)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `sin_dheader_t` | struct (extern) | BSP file header with lump table; used as `header` / `outheader` |
| `sin_dmodel_t` | struct (extern) | Per-model bounds, headnode, face range |
| `sin_dvis_t` | struct (extern) | Visibility data header: `numclusters`, `bitofs` table |
| `winding_t` | struct (from l_poly.h) | Convex polygon; used for brush-side geometry tests |
| `sin_lump_t` | struct (extern) | Lump descriptor: `fileofs`, `filelen` |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `sin_dmodels` … `sin_lightinfo` | typed pointers + int counts | global | One pointer+count pair per BSP lump section; dynamically allocated by `Sin_AllocMaxBSP` |
| `sin_dbrushsidetextured` | `char[SIN_MAX_MAP_BRUSHSIDES]` | global | Tracks which brush sides have had a valid texinfo assigned |
| `sin_dpop` | `byte[256]` | global | Stores the POP lump blob |
| `sin_bspallocated` | `int` | global | Allocation flag (unused after setting) |
| `sin_allocatedbspmem` | `int` | global | Running total of bytes allocated for BSP data |
| `header` | `sin_dheader_t *` | file-static | Pointer to the current BSP header during load/write |
| `outheader` | `sin_dheader_t` | file-static | Stack-allocated output header for `Sin_WriteBSPFile` |
| `wadfile` | `FILE *` | file-static | Output file handle during `Sin_WriteBSPFile` |

## Key Functions / Methods

### Sin_AllocMaxBSP
- **Signature:** `void Sin_AllocMaxBSP(void)`
- **Purpose:** Heap-allocates zeroed buffers for every BSP lump at their maximum allowed sizes.
- **Inputs:** None
- **Outputs/Return:** None (sets all global lump pointers)
- **Side effects:** Allocates heap memory tracked in `sin_allocatedbspmem`; logs allocation size.
- **Calls:** `GetClearedMemory`, `Log_Print`, `PrintMemorySize`
- **Notes:** Must be called before any load or write. `sin_dvis` is aliased to `sin_dvisdata`.

### Sin_FreeMaxBSP
- **Signature:** `void Sin_FreeMaxBSP(void)`
- **Purpose:** Frees all dynamically allocated BSP lump buffers and resets counts and pointers.
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Frees heap; logs freed size; zeroes `sin_allocatedbspmem`.
- **Calls:** `FreeMemory`, `Log_Print`, `PrintMemorySize`

### Sin_SwapBSPFile
- **Signature:** `void Sin_SwapBSPFile(qboolean todisk)`
- **Purpose:** Byte-swaps all numeric fields in every lump array between host and little-endian disk order.
- **Inputs:** `todisk` — controls direction for the visibility cluster count read.
- **Outputs/Return:** None (modifies globals in-place)
- **Side effects:** Mutates all global lump arrays; `#ifdef SIN` branches handle SiN-specific fields.
- **Calls:** `LittleLong`, `LittleShort`, `LittleFloat`, `LittleUnsignedShort`, `LittleUnsigned`
- **Notes:** Called once after load (todisk=false) and once before write (todisk=true); data should not be accessed after a todisk=true call.

### Sin_LoadBSPFile
- **Signature:** `void Sin_LoadBSPFile(char *filename, int offset, int length)`
- **Purpose:** Loads a SiN BSP file from disk, validates the header, copies all lumps into global arrays, then byte-swaps to host order.
- **Inputs:** `filename`, byte `offset` and `length` within the file (for embedded BSPs).
- **Outputs/Return:** None
- **Side effects:** Populates all global lump arrays; frees the raw file buffer after copying.
- **Calls:** `LoadFile`, `LittleLong`, `Sin_CopyLump`, `Error`, `FreeMemory`, `Sin_SwapBSPFile`
- **Notes:** Validates `SIN_BSPHEADER` / `SINGAME_BSPHEADER` and version; errors on mismatch.

### Sin_WriteBSPFile
- **Signature:** `void Sin_WriteBSPFile(char *filename)`
- **Purpose:** Byte-swaps global arrays to disk order and writes a complete SiN BSP file.
- **Inputs:** `filename` — output path.
- **Outputs/Return:** None
- **Side effects:** Modifies global arrays (byte-swapped in place); writes to disk; leaves data in disk-byte-order after return.
- **Calls:** `Sin_SwapBSPFile`, `SafeOpenWrite`, `SafeWrite`, `Sin_AddLump`, `fseek`, `fclose`

### Sin_CopyLump
- **Signature:** `int Sin_CopyLump(int lump, void *dest, int size[, int maxsize])`
- **Purpose:** Copies a raw lump from the loaded header into a destination buffer; validates alignment and bounds.
- **Inputs:** `lump` index, destination pointer, element `size`, and (SIN build) `maxsize`.
- **Outputs/Return:** Number of elements copied.
- **Side effects:** `memcpy` into destination.
- **Calls:** `Error`, `memcpy`

### Sin_FaceOnWinding
- **Signature:** `float Sin_FaceOnWinding(sin_dface_t *face, winding_t *winding)`
- **Purpose:** Returns the area of overlap between a BSP face (expressed as a clipping polygon) and a given winding.
- **Inputs:** `face`, `winding`
- **Outputs/Return:** Overlapping area (float); 0 if none.
- **Side effects:** Allocates and frees a temporary winding copy.
- **Calls:** `CopyWinding`, `VectorNegate`, `VectorSubtract`, `CrossProduct`, `VectorNormalize`, `DotProduct`, `ChopWindingInPlace`, `WindingArea`, `FreeWinding`

### Sin_BrushSideWinding
- **Signature:** `winding_t *Sin_BrushSideWinding(sin_dbrush_t *brush, sin_dbrushside_t *baseside)`
- **Purpose:** Constructs a winding for a brush side by starting with a base plane winding and clipping it by all other brush side planes.
- **Inputs:** `brush`, `baseside`
- **Outputs/Return:** Resulting `winding_t *` (caller must free), or NULL if fully clipped.
- **Side effects:** Allocates winding memory.
- **Calls:** `BaseWindingForPlane`, `DotProduct`, `fabs`, `ChopWindingInPlace`

### Sin_FixTextureReferences
- **Signature:** `void Sin_FixTextureReferences(void)`
- **Purpose:** For each brush side lacking a valid texture, finds a co-planar BSP face and copies its `texinfo`.
- **Inputs:** None (operates on globals)
- **Side effects:** Modifies `sin_dbrushsides[*].texinfo`; writes `sin_dbrushsidetextured`.
- **Calls:** `Sin_HintSkipBrush`, `Sin_BrushSideWinding`, `WindingIsTiny`, `WindingError`, `WindingArea`, `Sin_FaceOnWinding`, `FreeWinding`

### Sin_CompressVis / Sin_DecompressVis
- **Notes:** Standard run-length encoding (zero-byte RLE) for PVS bitsets. Row size derived from `sin_dvis->numclusters`. `DecompressVis` calls `Error` on a zero repeat count.

### Sin_ParseEntities / Sin_UnparseEntities
- **Notes:** Round-trip entity lump string ↔ `entities[]` array using the script parser. `UnparseEntities` errors if entity text overflows `SIN_MAX_MAP_ENTSTRING`.

## Control Flow Notes
This file is a standalone tool-side module (BSPC), not part of the runtime engine. It has no frame/update loop. Typical call order: `Sin_AllocMaxBSP` → `Sin_LoadBSPFile` → processing → `Sin_WriteBSPFile` → `Sin_FreeMaxBSP`.

## External Dependencies
- `l_cmd.h` — `LoadFile`, `SafeOpenWrite`, `SafeWrite`, `Error`, `LittleLong`, `LittleShort`, `LittleFloat`, `LittleUnsigned*`, `StripTrailing`
- `l_mem.h` — `GetClearedMemory`, `GetMemory`, `FreeMemory`, `PrintMemorySize`
- `l_log.h` — `Log_Print`
- `l_poly.h` — `winding_t`, `CopyWinding`, `FreeWinding`, `BaseWindingForPlane`, `ChopWindingInPlace`, `WindingArea`, `WindingError`, `WindingIsTiny` (declared extern in file)
- `l_bsp_ent.h` — `entities[]`, `num_entities`, `epair_t`, `entity_t`, `ParseEntity`
- `l_bsp_sin.h` — all `sin_d*` struct type definitions, lump constants (`SIN_LUMP_*`, `SIN_MAX_MAP_*`), surface flags
- `../botlib/l_script.h` — `LoadScriptMemory`, `SetScriptFlags`, `FreeScript`
- `WindingIsTiny` — declared forward in this file, defined elsewhere
