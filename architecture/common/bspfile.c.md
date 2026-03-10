# common/bspfile.c

## File Purpose
Implements the BSP (Binary Space Partitioning) file I/O layer for Quake III Arena's offline tools (q3map, bspc, etc.). It owns all global BSP lump arrays, handles loading and writing BSP files with byte-order swapping, and provides entity key/value parsing utilities.

## Core Responsibilities
- Define and own all global BSP lump data arrays (geometry, visibility, lighting, entities, etc.)
- Load a BSP file from disk, copy each lump into its global array, and byte-swap all data
- Write global BSP arrays back to disk as a well-formed BSP file with a corrected header
- Byte-swap individual BSP structs for endian portability
- Parse the raw entity string (`dentdata`) into an in-memory `entity_t` array
- Serialize the `entity_t` array back into the `dentdata` string
- Provide key/value accessors (`ValueForKey`, `SetKeyValue`, `FloatForKey`, `GetVectorForKey`) for entity manipulation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `epair_t` | struct (typedef) | Singly-linked key/value string pair for a single entity attribute |
| `entity_t` | struct (typedef) | Parsed map entity: origin, brush/patch lists, first draw surface index, and epair chain |
| `dheader_t` | struct (defined in qfiles.h) | BSP file header with ident, version, and lump directory |
| `lump_t` | struct (defined in qfiles.h) | File offset + length descriptor for one BSP lump |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dmodels` / `nummodels` | `dmodel_t[]` / `int` | global | Sub-model (brush model) lump |
| `dshaders` / `numShaders` | `dshader_t[]` / `int` | global | Shader reference lump |
| `dentdata` / `entdatasize` | `char[]` / `int` | global | Raw entity string lump |
| `dleafs` / `numleafs` | `dleaf_t[]` / `int` | global | BSP leaf lump |
| `dplanes` / `numplanes` | `dplane_t[]` / `int` | global | Plane lump |
| `dnodes` / `numnodes` | `dnode_t[]` / `int` | global | BSP node lump |
| `dleafsurfaces` / `numleafsurfaces` | `int[]` / `int` | global | Leaf-to-surface index lump |
| `dleafbrushes` / `numleafbrushes` | `int[]` / `int` | global | Leaf-to-brush index lump |
| `dbrushes` / `numbrushes` | `dbrush_t[]` / `int` | global | Brush lump |
| `dbrushsides` / `numbrushsides` | `dbrushside_t[]` / `int` | global | Brush side lump |
| `lightBytes` / `numLightBytes` | `byte[]` / `int` | global | Lightmap lump (raw bytes) |
| `gridData` / `numGridPoints` | `byte[]` / `int` | global | Light grid lump (8 bytes/point) |
| `visBytes` / `numVisBytes` | `byte[]` / `int` | global | Visibility data lump |
| `drawVerts` / `numDrawVerts` | `drawVert_t[]` / `int` | global | Draw vertex lump |
| `drawIndexes` / `numDrawIndexes` | `int[]` / `int` | global | Draw index lump |
| `drawSurfaces` / `numDrawSurfaces` | `dsurface_t[]` / `int` | global | Draw surface lump |
| `dfogs` / `numFogs` | `dfog_t[]` / `int` | global | Fog volume lump |
| `entities` / `num_entities` | `entity_t[]` / `int` | global | Parsed entity array |

## Key Functions / Methods

### SwapBlock
- **Signature:** `void SwapBlock( int *block, int sizeOfBlock )`
- **Purpose:** Byte-swaps every 32-bit word in an arbitrary memory block for endian conversion.
- **Inputs:** Pointer to block, byte size of block.
- **Outputs/Return:** void; modifies block in place.
- **Side effects:** Overwrites the input buffer.
- **Calls:** `LittleLong`
- **Notes:** Only correct when all fields in the block are 32-bit; mixed-width structs (e.g. `drawVert_t`) are handled separately.

### SwapBSPFile
- **Signature:** `void SwapBSPFile( void )`
- **Purpose:** Byte-swaps all loaded BSP lumps in the global arrays. Called after load and before write.
- **Inputs:** None (operates on globals).
- **Outputs/Return:** void.
- **Side effects:** Mutates every global lump array in place.
- **Calls:** `SwapBlock`, `LittleLong`, `LittleFloat`
- **Notes:** `drawVert_t` and fog/shader lumps need field-by-field swapping due to mixed types; colors (`byte[4]`) are skipped. Visibility header words (cluster count, row size) are swapped individually.

### CopyLump
- **Signature:** `int CopyLump( dheader_t *header, int lump, void *dest, int size )`
- **Purpose:** Copies a single BSP lump from the in-memory file image into a destination buffer and returns the element count.
- **Inputs:** File header pointer, lump index, destination buffer, element size in bytes.
- **Outputs/Return:** Number of elements copied (`length / size`).
- **Side effects:** `memcpy` into `dest`; calls `Error` if lump size is not a multiple of element size.
- **Calls:** `memcpy`, `Error`

### LoadBSPFile
- **Signature:** `void LoadBSPFile( const char *filename )`
- **Purpose:** Loads a `.bsp` file into all global lump arrays and byte-swaps the data.
- **Inputs:** Path to BSP file.
- **Outputs/Return:** void; populates all global arrays.
- **Side effects:** Allocates then frees the raw file buffer; calls `Error` on bad ident/version; calls `SwapBSPFile`.
- **Calls:** `LoadFile`, `SwapBlock`, `CopyLump`, `Error`, `free`, `SwapBSPFile`

### WriteBSPFile
- **Signature:** `void WriteBSPFile( const char *filename )`
- **Purpose:** Writes all global BSP arrays to a new BSP file, re-swapping data for on-disk byte order.
- **Inputs:** Output path.
- **Outputs/Return:** void.
- **Side effects:** Creates/overwrites file; mutates global arrays via `SwapBSPFile` (file is not usable in-process after this call); seeks back to rewrite the header after all lumps are written.
- **Calls:** `SwapBSPFile`, `SafeOpenWrite`, `SafeWrite`, `AddLump`, `fseek`, `fclose`
- **Notes:** `AddLump` writes data padded to a 4-byte boundary; the header is written twice (placeholder then corrected).

### AddLump
- **Signature:** `void AddLump( FILE *bspfile, dheader_t *header, int lumpnum, const void *data, int len )`
- **Purpose:** Writes one lump to the BSP file and records its offset/length in the header.
- **Inputs:** Open file handle, header to update, lump index, data pointer, byte length.
- **Side effects:** Advances file position; writes `(len+3)&~3` bytes (4-byte aligned).
- **Calls:** `ftell`, `LittleLong`, `SafeWrite`

### ParseEntities / UnparseEntities
- **Signature:** `void ParseEntities(void)` / `void UnparseEntities(void)`
- **Purpose:** Convert between the raw `dentdata` string and the `entities[]` / `num_entities` globals.
- **Side effects:** `ParseEntities` resets `num_entities` and calls `ParseFromMemory`+`ParseEntity` in a loop; `UnparseEntities` rebuilds `dentdata` and updates `entdatasize`. Both `malloc` epair nodes.
- **Calls:** `ParseFromMemory`, `ParseEntity`, `ParseEpair`, `GetToken`, `Error`, `copystring`, `malloc`, `strcat`, `sprintf`, `StripTrailing`

### ValueForKey / SetKeyValue / FloatForKey / GetVectorForKey
- **Notes:** Trivial linear-search accessors over an entity's `epair_t` linked list. `SetKeyValue` inserts a new epair if key not found. `GetVectorForKey` uses `sscanf` into doubles for `vec_t`-size independence.

## Control Flow Notes
This file is used exclusively by offline map-compilation tools (q3map, bspc). It has no role in the runtime engine. Typical usage: `LoadBSPFile` at tool startup → manipulate global arrays → `WriteBSPFile` at completion. `ParseEntities`/`UnparseEntities` are called as needed to edit entity data between load and write.

## External Dependencies
- `cmdlib.h` — `LoadFile`, `SafeWrite`, `SafeOpenWrite`, `Error`, `copystring`, `LittleLong`, `LittleFloat`, `qboolean`
- `mathlib.h` — `vec_t`, `vec3_t`
- `bspfile.h` — BSP struct types (`dheader_t`, `dmodel_t`, `dleaf_t`, `drawVert_t`, `dsurface_t`, etc.), lump index constants, `MAX_MAP_*` limits (all defined via `qfiles.h` / `surfaceflags.h`)
- `scriplib.h` — `ParseFromMemory`, `GetToken`, `token` global (used by `ParseEntity`/`ParseEpair`)
- `GetLeafNums` — declared but never called in this file; defined elsewhere
