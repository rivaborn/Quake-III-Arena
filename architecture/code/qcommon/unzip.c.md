# code/qcommon/unzip.c

## File Purpose
A self-contained ZIP decompression library, adapted from zlib 1.1.3 / minizip 0.15 and embedded directly into the Quake III engine. It provides the `unzFile` API used by `files.c` to read game assets (`.pk3` files are ZIP archives). The entire zlib inflate pipeline — block processing, Huffman tree building, code decoding, Adler-32 checksumming — is inlined here as a single translation unit.

## Core Responsibilities
- Open, enumerate, and close ZIP archives via `unzOpen`/`unzClose`
- Navigate the ZIP central directory to locate specific files by name
- Open a specific file entry within a ZIP for streaming read (`unzOpenCurrentFile`)
- Decompress stored (method 0) or deflated (method 8) entries into caller-supplied buffers
- Provide the full zlib inflate state machine: `inflate`, `inflate_blocks`, `inflate_codes`, `inflate_fast`
- Build Huffman decode trees for dynamic and fixed deflate blocks (`huft_build`, `inflate_trees_*`)
- Replace zlib's `malloc`/`free` with Q3's `Z_Malloc`/`Z_Free` via `zcalloc`/`zcfree`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `unz_s` | struct | Per-open-ZIP state: FILE handle, central dir position, current-file cursor, read-in-progress pointer |
| `file_in_zip_read_info_s` | struct | Per-open-entry state: compressed/uncompressed byte counts, zlib stream, read buffer |
| `unz_file_info` | struct | Public metadata for one ZIP entry (sizes, CRC, dates, flags) |
| `unz_file_info_internal` | struct | Internal metadata: raw offset to local file header |
| `unz_global_info` | struct | ZIP-level metadata: entry count, comment size |
| `z_stream` / `z_streamp` | struct/typedef | zlib stream with I/O pointers, allocator hooks, running totals |
| `inflate_blocks_state` | struct | State for one deflate block: mode FSM, sliding window, bit buffer, Huffman tables |
| `inflate_codes_state` | struct | State for literal/length/distance code decoding within a block |
| `inflate_huft` | struct | One entry in a multi-level Huffman lookup table (exop, bits, base) |
| `inflate_block_mode` | enum | Block-level FSM states: TYPE, LENS, STORED, TABLE, BTREE, DTREE, CODES, DRY, DONE, BAD |
| `inflate_codes_mode` | enum | Code-level FSM states: START, LEN, LENEXT, DIST, DISTEXT, COPY, LIT, WASH, END, BADCODE |
| `inflate_mode` | enum | Top-level zlib stream FSM: imMETHOD, imFLAG, imBLOCKS, imCHECK*, imDONE, imBAD |
| `internal_state` | struct | Top-level zlib decompressor state: mode, nowrap flag, wbits, pointer to blocks state |
| `tm_unz` | struct | DOS date/time broken out into year/month/day/hour/min/sec fields |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `inflate_mask[17]` | `static uInt[]` | static | Bit-mask table for extracting N-bit fields from bit buffer |
| `fixed_bl`, `fixed_bd` | `static uInt` | static | Fixed Huffman tree bit depths (9, 5) |
| `fixed_tl[]`, `fixed_td[]` | `static inflate_huft[]` | static | Pre-built fixed literal/length and distance Huffman tables |
| `cplens[31]`, `cplext[31]` | `static const uInt[]` | static | Copy-length base values and extra-bit counts for codes 257–285 |
| `cpdist[30]`, `cpdext[30]` | `static const uInt[]` | static | Distance base values and extra-bit counts for distance codes |
| `border[19]` | `static const uInt[]` | static | Bit-length code reordering table from PKZIP spec |
| `inflate_copyright[]` | `static const char[]` | static | zlib attribution string embedded in binary |

## Key Functions / Methods

### unzOpen
- **Signature:** `extern unzFile unzOpen(const char *path)`
- **Purpose:** Opens a ZIP file, locates and validates the central directory, allocates and returns a `unz_s` handle.
- **Inputs:** `path` — filesystem path to `.pk3`/`.zip` file.
- **Outputs/Return:** Opaque `unzFile` handle, or `NULL` on failure.
- **Side effects:** `fopen`, `ALLOC` (→ `Z_Malloc`), leaves file handle open inside returned state.
- **Calls:** `unzlocal_SearchCentralDir`, `unzlocal_getLong`, `unzlocal_getShort`, `fseek`, `fclose`
- **Notes:** Multi-disk (spanning) ZIPs are rejected. Does not call `unzGoToFirstFile`; caller must navigate.

### unzReOpen
- **Signature:** `extern unzFile unzReOpen(const char *path, unzFile file)`
- **Purpose:** Creates a second independent handle to the same ZIP by copying an existing `unz_s` and opening a fresh FILE pointer. Used for concurrent reads in `files.c`.
- **Inputs:** `path`, source `unzFile`.
- **Outputs/Return:** New `unzFile` handle or `NULL`.
- **Side effects:** `fopen`, `ALLOC`, `Com_Memcpy`.

### unzLocateFile
- **Signature:** `extern int unzLocateFile(unzFile file, const char *szFileName, int iCaseSensitivity)`
- **Purpose:** Linear scan of all central-directory entries to find a named file; sets it as the current file on success.
- **Inputs:** Open ZIP handle, target filename, case-sensitivity mode (0=OS default, 1=sensitive, 2=insensitive).
- **Outputs/Return:** `UNZ_OK` if found (current file updated); `UNZ_END_OF_LIST_OF_FILE` if not found.
- **Side effects:** Modifies `s->num_file` and `s->pos_in_central_dir`; restores them on failure.
- **Calls:** `unzGoToFirstFile`, `unzGoToNextFile`, `unzGetCurrentFileInfo`, `unzStringFileNameCompare`

### unzOpenCurrentFile
- **Signature:** `extern int unzOpenCurrentFile(unzFile file)`
- **Purpose:** Prepares the current ZIP entry for reading: validates the local file header, allocates a read buffer and (for deflated entries) initialises the zlib inflate stream.
- **Inputs:** Open ZIP handle with a valid current file.
- **Outputs/Return:** `UNZ_OK` or error code.
- **Side effects:** Allocates `file_in_zip_read_info_s` and 64 KB read buffer; calls `inflateInit2` for compressed entries.
- **Calls:** `unzlocal_CheckCurrentFileCoherencyHeader`, `inflateInit2`, `unzCloseCurrentFile` (if already open)

### unzReadCurrentFile
- **Signature:** `extern int unzReadCurrentFile(unzFile file, void *buf, unsigned len)`
- **Purpose:** Reads up to `len` uncompressed bytes from the currently open entry, decompressing on the fly.
- **Inputs:** Open ZIP handle (entry opened), output buffer, byte count.
- **Outputs/Return:** Bytes read (>0), 0 at EOF, negative error code on failure.
- **Side effects:** Advances file position; calls `fseek`+`fread` to pull compressed chunks; calls `inflate` for deflated data.
- **Calls:** `fseek`, `fread`, `inflate`
- **Notes:** CRC32 verification is commented out. For stored entries, does a manual byte-copy loop instead of `memcpy`.

### inflate
- **Signature:** `static int inflate(z_streamp z, int f)`
- **Purpose:** Top-level zlib inflate FSM; processes the zlib wrapper (method byte, flags, Adler-32 check) and dispatches to `inflate_blocks`.
- **Inputs:** zlib stream, flush mode.
- **Outputs/Return:** `Z_OK`, `Z_STREAM_END`, `Z_DATA_ERROR`, etc.
- **Calls:** `inflate_blocks`, `inflate_blocks_reset`

### inflate_blocks
- **Signature:** `int inflate_blocks(inflate_blocks_statef *s, z_streamp z, int r)`
- **Purpose:** Decodes a sequence of deflate blocks (stored, fixed-Huffman, dynamic-Huffman) using a state machine driven by the `inflate_block_mode` enum.
- **Calls:** `inflate_trees_fixed`, `inflate_trees_dynamic`, `inflate_trees_bits`, `inflate_codes_new`, `inflate_codes`, `inflate_codes_free`, `inflate_flush`

### huft_build
- **Signature:** `static int huft_build(uInt *b, uInt n, uInt s, const uInt *d, const uInt *e, inflate_huft **t, uInt *m, inflate_huft *hp, uInt *hn, uInt *v)`
- **Purpose:** Constructs a multi-level Huffman lookup table from an array of code lengths.
- **Outputs/Return:** `Z_OK`, `Z_BUF_ERROR` (incomplete table), `Z_DATA_ERROR` (over-subscribed), `Z_MEM_ERROR`.
- **Notes:** Allocates table entries from pre-allocated `hp` pool; does not call `malloc` directly.

### zcalloc / zcfree
- **Signature:** `voidp zcalloc(voidp opaque, unsigned items, unsigned size)` / `void zcfree(voidp opaque, voidp ptr)`
- **Purpose:** Bridge zlib's pluggable allocator to Q3's `Z_Malloc`/`Z_Free`.
- **Calls:** `Z_Malloc`, `Z_Free`

### Notes on minor helpers
- `unzlocal_getShort` / `unzlocal_getLong`: read little-endian 2/4-byte values from FILE using `LittleShort`/`LittleLong`.
- `adler32`: computes running Adler-32 checksum using unrolled DO16 macros.
- `inflate_flush`: copies bytes from the sliding window ring buffer to the output stream.
- `inflate_fast`: optimised hot path for literal/length/distance decoding when ≥258 output bytes and ≥10 input bytes are available.

## Control Flow Notes
This file is used exclusively during **init / asset loading**. `files.c` calls `unzOpen` at filesystem startup to index `.pk3` files, then calls `unzLocateFile` + `unzOpenCurrentFile` + `unzReadCurrentFile` on demand for each asset read. There is no per-frame involvement. The inflate pipeline is entirely re-entrant (no global mutable state beyond the static lookup tables).

## External Dependencies
- `../client/client.h` → transitively pulls in `q_shared.h`, `qcommon.h` (for `Com_Memcpy`, `Com_Memset`, `Z_Malloc`, `Z_Free`, `LittleShort`, `LittleLong`)
- `unzip.h` — declares the public `unzFile` API types and error codes consumed by `files.c`
- **Defined elsewhere:** `Z_Malloc`, `Z_Free`, `Com_Memcpy`, `Com_Memset`, `LittleShort`, `LittleLong`, `Sys_Error` (debug only)
- No platform I/O beyond standard C `FILE*` (`fopen`, `fread`, `fseek`, `ftell`, `fclose`)
