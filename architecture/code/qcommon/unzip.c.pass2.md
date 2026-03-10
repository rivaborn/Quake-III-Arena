# code/qcommon/unzip.c — Enhanced Analysis

## Architectural Role
This file provides the **foundational asset-loading pipeline** for Quake III's virtual filesystem layer (`files.c`). By embedding zlib's entire inflate codec + minizip's ZIP parsing, it eliminates external runtime dependencies while maintaining single-file simplicity. All game assets (textures, models, scripts, maps) stream through this module during engine startup and on-demand file reads; there is no per-frame decompression overhead. The self-contained architecture—zlib wrapper validation → block-level FSM → Huffman tree building → code decoding—mirrors early-2000s game engines before package managers became standard.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/qcommon/files.c`**: Core virtual filesystem (`FS_LoadZipFile`, asset enumeration, file searching) calls all public unzFile API functions
- **Client/Server**: Indirectly, via `ri.FS_ReadFile` and `trap_FS_*` syscalls during asset loading (init-time only; no runtime hot-path involvement)

### Outgoing (what this file depends on)
- **`code/qcommon/qcommon.h`** (transitively via `client.h`): Exports `Z_Malloc`, `Z_Free`, `Com_Memcpy`, `Com_Memset`, `LittleShort`, `LittleLong`
- **Standard C `<stdio.h>`**: FILE I/O primitives (`fopen`, `fread`, `fseek`, `ftell`, `fclose`) for all ZIP metadata and compressed block reads
- **No other engine subsystems**: Pure utility; does not call renderer, server, or game VM code

## Design Patterns & Rationale

| Pattern | Evidence | Rationale |
|---------|----------|-----------|
| **Embedded library** | All zlib 1.1.3 + minizip 0.15 inlined into single `.c` file; no `.a`/`.so` dependency | Simplifies deployment, avoids version conflicts, enables platform-agnostic builds (1998–2005 era: not yet standard to vendor via git submodules) |
| **Pluggable allocator** | `zcalloc`/`zcfree` bridge zlib's `z_stream.zalloc`/`z_freep` to Q3's `Z_Malloc`/`Z_Free` | Ensures all decompression buffers (Huffman tables, sliding window, etc.) account toward zone memory budget and survive `Com_Error` longjmp via hunk persistence |
| **Pre-computed Huffman tables** | `fixed_tl[]`, `fixed_td[]` statically built at compile-time for RFC 1951 "fixed block" case | Avoids tree-building overhead for 70% of real-world ZIPs; `huft_build` only called for "dynamic block" deflate streams |
| **Streaming decompression** | 64 KB read buffer + circular inflate state; single-pass entry open; no buffering entire compressed payload | Enables loading of large `.pk3` files without proportional memory spike |
| **Bit-buffer FSM** | `inflate_blocks` mode enum driven by input bit stream; `inflate_codes` state machine for literal/length/distance decoding | Low-level zlib spec compliance; each mode reads only enough bits needed before yielding control |
| **ZIP metadata caching** | Central directory parsed into `unz_file_info` array on `unzOpen`; `unzLocateFile` does linear scan | Trade-off: Fast random file lookup within a ZIP, but O(n) cost; acceptable because ZIP file counts typically ≤1000 |

## Data Flow Through This File

```
Asset Load Request (e.g., "models/players/grunt/model.md3")
    ↓
files.c: FS_LoadZipFile("pak0.pk3")
    ↓
unzOpen(path)
  • fopen + scan backwards for central directory (EOCD)
  • unzlocal_SearchCentralDir + unzlocal_getLong
  • Populate unz_s{ cdir_pos, num_file, current_file_num }
    ↓
unzLocateFile("models/players/...")
  • Linear scan of central directory entries
  • Match filename; set s->pos_in_central_dir
    ↓
unzOpenCurrentFile()
  • Validate local file header (CRC, sizes)
  • Allocate 64 KB read buffer + file_in_zip_read_info_s
  • If compressed (method==8): inflateInit2(z, -MAX_WBITS)
    ↓
unzReadCurrentFile(buf, len) × N [loop until EOF]
  • fseek/fread pulls compressed chunks
  • inflate(z_stream, Z_SYNC_FLUSH) decompresses into caller buffer
  • Adler-32 running checksum (CRC verification commented out)
  • Returns uncompressed byte count
    ↓
unzCloseCurrentFile()
  • inflateEnd + free 64 KB buffer + free read state
    ↓
unzClose(file)
  • fclose underlying FILE; free unz_s
```

**Key state transitions:**
- `inflate_mode` FSM: `imMETHOD` → `imFLAG` → `imBLOCKS` (where blocks FSM iterates) → `imCHECK*` → `imDONE`
- `inflate_block_mode` FSM per block: `TYPE` → (`LENS`→`CODES`) or `STORED` → `DRY` → resets for next block
- `inflate_codes_mode` FSM: `START` → `LEN`/`LENEXT` → `DIST`/`DISTEXT` → `COPY`/`LIT` → `WASH` → `END`

## Learning Notes

### Era-Specific Design (Pre-Package-Management)
- **Monolithic embedded library** was industry standard in 1998–2005: modern practice (git submodule, ConanIO, vcpkg) did not exist
- **Streaming decompression** necessary because RAM was scarce (128 MB–512 MB systems); buffering a 50 MB texture would fail
- **Rejection of multi-disk ZIPs** reflects era of distribution: CD-ROM spanning was common in commercial software, but Q3's asset format rejected it upfront (design decision)

### Modern Engine Divergence
- **Embedded zlib** vs. linked system library: Modern engines (Unreal, Unity) link against OS-provided zlib or custom formats (UASSET, proprietary compression)
- **No encryption** (despite PKZIP 2.04g support): Modern engines use content-addressed blob stores with separate encryption (TLS for downloads, hardware-accelerated AES for storage)
- **No async decompression**: All I/O is synchronous on main thread; modern engines decompress on background worker threads with job system

### Key Decompression Insights
- **Huffman decoding**: `huft_build` constructs multi-level lookup tables to decode variable-length codes in O(1) lookups per symbol
- **Sliding window ring buffer**: `inflate_flush` copies literal/match data back to caller via a 32 KB circular buffer (RFC 1951 §3.2.3)
- **Two-pass block structure**: Dynamic Huffman blocks store bit-length code sequence in compressed form (HCLEN codes) which must be decoded to then decode the literal/length/distance alphabets
- **Fast path optimization**: `inflate_fast` avoids per-byte state updates when ≥258 output bytes and ≥10 input bytes available (common case in large blocks)

## Potential Issues

- **CRC32 verification disabled** (line comments show checksum computed but not validated): Allows detection of random corruption but not deliberate tampering. Modern engines validate via secure hashes (SHA-256).
- **No truncation detection**: If ZIP central directory is malformed or file list incomplete, `unzReadCurrentFile` may read beyond actual entry bounds.
- **Memory leaks in error paths**: `unzOpenCurrentFile` allocates buffer and initializes inflate state; if `inflateInit2` fails, buffer freed via cleanup but caller must verify `unzOpenCurrentFile` return code before calling `unzReadCurrentFile`.
- **64 KB per-file overhead**: Each `unzOpenCurrentFile` allocates 65 KB regardless of actual compressed block size; multiple concurrent opens scale linearly (acceptable for asset loading, problematic for streaming IO).
- **Case-sensitivity on Windows**: `unzLocateFile` with `iCaseSensitivity=0` defers to OS default, which is case-insensitive on Windows but case-sensitive on Linux/Mac; can cause silent asset misses if path casing inconsistent.
