# libs/pak/unzip.h — Enhanced Analysis

## Architectural Role

This header defines the public C API for ZIP file reading, enabling the engine's virtual filesystem (`code/qcommon/files.c`) to transparently load `.pk3` archive files (which are ZIP format) alongside directory-based asset trees. By abstracting ZIP decompression through a handle-based interface, it allows the engine to treat archived and loose assets uniformly, supporting the modular content delivery model where game data (textures, models, scripts, shaders) may be packaged into `.pk3` files with priority-based merging and optional CRC validation for server purity checks.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/qcommon/files.c`** — The virtual filesystem layer calls unzip functions to enumerate and extract files from `.pk3` archives during asset loading (indirectly, via `code/qcommon/unzip.c` wrapper)
- **`libs/pak/unzip.cpp`** — The implementation file; provides the backing code for all declared functions
- **`q3map` / `q3radiant` tools** — May use unzip via `libs/pak/pakstuff.cpp` for reading base `.pk3` files during map compilation or editing
- **Game initialization** — At engine startup, the FS layer scans directories and loads `.pk3` files to populate the searchpath priority stack

### Outgoing (what this file depends on)

- **zlib** — Uses the `z_stream` structure and zlib decompression callbacks (`alloc_func`, `free_func`) for on-the-fly deflate decompression of file content
- **Platform I/O** — Indirectly depends on `FILE*` and standard C file operations (implicitly linked at implementation level)
- **Memory allocators** — Custom allocation hooks allow the engine to substitute its own zone/hunk allocators instead of malloc/free

## Design Patterns & Rationale

**Opaque Handle Pattern** — `unzFile` is defined as either a strict struct pointer (if `STRICTUNZIP` defined) or a void pointer (default). This isolates callers from internal state (`unz_s`) and allows future implementation changes without ABI breaks.

**State Machine for Navigation** — The triple of `unzGoToFirstFile` → `unzGoToNextFile` → `unzGetCurrentFileInfo` mirrors a classic ZIP central-directory walk. This allows lazy enumeration without pre-loading the entire directory structure.

**Dual Decompression Strategy** — The header includes both compressed metadata (stored plainly) and a full `z_stream` structure embedded in `file_in_zip_read_info_s`, allowing on-demand decompression only when the file content is read via `unzReadCurrentFile`, not when metadata is fetched.

**CRC Validation** — Storage of both `crc32_wait` (expected) and computed `crc32` in the read-info struct enables post-decompression integrity checking, critical for ensuring map/asset integrity in networked scenarios where pure-server checks reject mismatched `.pk3` files.

**Custom Allocators** — The `alloc_func`/`free_func` hooks in `z_stream` allow zlib to be fed back through Quake's hunk/zone allocators, avoiding malloc fragmentation and enabling unified memory profiling.

## Data Flow Through This File

1. **Initialization** — `unzOpen(path)` opens a `.pk3` file, reads the ZIP end-of-central-directory record, validates format, and returns an opaque handle
2. **Global Metadata** — `unzGetGlobalInfo()` extracts entry count and comment size without enumerating files
3. **Directory Traversal** — `unzGoToFirstFile()` → loop with `unzGoToNextFile()` allows the FS layer to enumerate every asset
4. **Per-File Metadata** — `unzGetCurrentFileInfo()` populates `unz_file_info` with CRC, size, timestamps, compression method for each asset
5. **Decompression Setup** — `unzOpenCurrentFile()` initializes a `z_stream` and allocates read buffers, transitioning to `file_in_zip_read_info_s` state
6. **Streaming Read** — `unzReadCurrentFile(buf, len)` pumps the input stream through zlib, incrementally decompressing to caller's buffer and accumulating CRC
7. **Validation** — `unzCloseCurrentFile()` compares computed vs. stored CRC; returns `UNZ_CRCERROR` if mismatch (prevents corrupted assets from reaching game)
8. **Cleanup** — `unzClose()` releases all buffers and the FILE handle

## Learning Notes

**ZIP Internals** — Developers using this will learn that ZIP archives store a central directory at the file's end (requiring a reverse scan to find it), with local headers preceding each file's raw data. This explains the position tracking variables (`pos_in_central_dir`, `pos_in_zipfile`, `central_pos`) scattered through the structures.

**Idiomatic C API Design** — The header demonstrates a pre-C99 idiomatic opaque-handle library: no dynamic memory exposure to the caller, all state hidden, and allocation hooks for custom memory management—a pattern rarely used in modern engines which favor RAII or GC.

**Case Sensitivity Tradeoff** — The `UNZ_CASESENSITIVE` / `UNZ_NOTCASESENSITIVE` / `UNZ_OSDEFAULTCASE` flags show awareness of cross-platform issues (case-preserving filesystems on Unix vs. case-insensitive on Windows), baked into the unzip layer rather than the FS abstraction above it.

**Zlib Integration** — The embedded `z_stream` structure and the `tm_unz` date struct (DOS-era format) show this code was written circa late 1990s / early 2000s, before modern compression libraries abstracted away bitstream details.

## Potential Issues

- **Type Safety** — Without `STRICTUNZIP` defined (the default), `unzFile` is a bare `void*` and type misuse goes undetected at compile time; easy source of subtle bugs if handles are confused with other pointers.
- **Buffer Sizing** — Functions like `unzGetGlobalComment()` and `unzGetCurrentFileInfo()` take buffer size parameters; no guard against caller-provided undersized buffers, relying entirely on caller discipline (pre-2000s C style).
- **Thread Safety** — No thread safety documentation; simultaneous reads on different file handles or navigation while reading likely causes state corruption.
- **Memory Leaks** — If `unzOpenCurrentFile()` succeeds but the caller never calls `unzCloseCurrentFile()`, the `z_stream` state and read buffer are never freed; no RAII cleanup on exception (though C doesn't have exceptions).
