# code/qcommon/unzip.h — Enhanced Analysis

## Architectural Role

This header is the **ZIP archive reader interface** for Quake III's virtual filesystem subsystem (qcommon). It provides the low-level API that `code/qcommon/files.c` uses to transparently mount and read `.pk3` files alongside directory-based assets during filesystem initialization and runtime asset lookups. The library is self-contained (minizip-derived) and exposes both public navigation/read functions and internal structures directly, enabling the VFS to efficiently iterate central directories and manage per-file decompression state across engine lifetime.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/qcommon/files.c`** (the core VFS layer) — calls `unzOpen()` at engine init to load `.pk3` files; enumerates entries with `unzGoToFirstFile()`/`unzGoToNextFile()` to build the in-memory pak/dir priority index; at runtime, uses `unzLocateFile()` to find assets by name, then `unzOpenCurrentFile()`/`unzReadCurrentFile()`/`unzCloseCurrentFile()` to decompress and stream them
- **Platform layer** (indirectly via `files.c`) — never directly; the VFS interface abstracts this
- **All subsystems that read assets** (renderer, cgame, server, game) — indirectly, through the qcommon VFS abstraction

### Outgoing (what this file depends on)
- **`<stdio.h>`** — `FILE*` opened/managed by caller; used directly in `unz_s` and `file_in_zip_read_info_s` for seek/read
- **zlib inflate API** — `struct internal_state` (forward-declared; defined in zlib internals); z_stream for `alloc_func`/`free_func` callbacks; error codes like `Z_ERRNO`
- **Caller must provide memory** — `z_stream.zalloc`/`zfree` callbacks (registered at open time) allocate decompression buffers; no malloc/free in unzip itself

## Design Patterns & Rationale

**Opaque Handle with Optional Strict Typing**
- `STRICTUNZIP` macro allows compile-time type checking of `unzFile` handles on Windows (via tagged empty struct), while defaulting to void* for portability
- Rationale: maximize compatibility across 1990s compilers while catching type errors where possible

**Iterator + Current State Cursor**
- `unzGoToFirstFile()` / `unzGoToNextFile()` / `unzLocateFile()` move a per-archive "current file" pointer in the central directory
- `unzGetCurrentFileInfo()` / `unzOpenCurrentFile()` / `unzReadCurrentFile()` operate on the current file
- Rationale: avoid passing positional state through the API; keeps internal navigation cache coherent

**Exposed Internal Structures**
- `unz_s`, `file_in_zip_read_info_s`, and `z_stream` are all public in the header
- Rationale: enables the VFS layer to directly inspect (e.g., `file_in_zip_read_info_s.file` pointer, `crc32` state) and possibly pre-allocate for performance; minizip design philosophy is "library for embedding"

**CRC Deferred Validation**
- CRC32 comparison only occurs at `unzCloseCurrentFile()`, after the entire entry has been decompressed
- Rationale: streaming read model; early detection of corruption is traded for simplicity

## Data Flow Through This File

1. **Archive Load Phase** (engine init via `code/qcommon/files.c`)
   - `unzOpen(path)` → reads ZIP end-of-central-dir, validates signature, allocates `unz_s` with FILE* and central-dir metadata
   - `unzGoToFirstFile()` → positions cursor at first central-dir entry
   - `unzGoToNextFile()` loop → each entry's metadata (`unz_file_info`) is queried, logged, and indexed by filename

2. **Runtime Asset Access**
   - `unzLocateFile(name, ...)` → binary/linear search through central-dir entries; sets current file pointer
   - `unzGetCurrentFileInfo(...)` → reads compression method, sizes, CRC from current entry header
   - `unzOpenCurrentFile()` → allocates `file_in_zip_read_info_s`, initializes zlib inflate stream, seeks to entry's local header
   - `unzReadCurrentFile(buf, len)` loop → decompresses chunk via `zlib inflate()`; maintains running CRC32
   - `unzCloseCurrentFile()` → validates final CRC32 against stored value; frees inflate state

3. **Memory/Resource State**
   - `unz_s` lifetime = archive handle lifetime (one per `.pk3`)
   - `file_in_zip_read_info_s` lifetime = per-file decompression session (created by `unzOpenCurrentFile`, destroyed by `unzCloseCurrentFile`)
   - `z_stream` is embedded in `file_in_zip_read_info_s`; allocations routed through caller-provided `zalloc`/`zfree`

## Learning Notes

**Portable 1990s Library Design**
- Minimal stdlib usage (only `<stdio.h>`); all types are explicit (`unsigned long`, `unsigned char`) for cross-platform consistency
- No dynamic allocation of core structures; caller provides memory callback hooks for zlib
- STRICTUNZIP opt-in type safety shows pragmatic backward-compat approach

**Minizip Inheritance**
- This is a thin wrapper over the zlib project's minizip reference implementation
- Exposes internal zlib/inflate state directly in the public header — unusual by modern standards, but practical for an embeddable library
- Date/time storage uses DOS format (`dosDate` in `unz_file_info`) — reflects ZIP spec's 1989 origin

**Contrast with Modern Engines**
- Modern engines (e.g., modern Unreal, Unity) typically hide archive I/O behind a fully abstracted `IArchiveReader` interface
- Q3's exposure of internal structures prioritized direct vfs.c access to `unz_s.file` pointer and compression state for efficiency
- No async I/O primitives; reads are fully synchronous (blocked until uncompressed bytes available)

**CRC Model Insight**
- `crc32` / `crc32_wait` split in `file_in_zip_read_info_s` allows incremental validation during streaming
- `unzCloseCurrentFile` returning `UNZ_CRCERROR` only after full read means truncated/damaged `.pk3` files can crash the engine mid-load, not during asset access

## Potential Issues

1. **No Thread Safety**
   - All structures (`unz_s`, current file cursor, `z_stream` state) assume single-threaded access
   - If the engine loads assets from multiple threads (renderer streaming thread + main loop), race conditions on `FILE*` seek/read are possible
   - Modern engines use per-thread decompression buffers or mutex protection; Q3 assumes single VFS owner

2. **CRC Validation Timing**
   - If `unzReadCurrentFile` is never called to completion (caller stops reading mid-entry), `unzCloseCurrentFile` won't detect CRC mismatch
   - Corrupted `.pk3` entries might silently return partial/garbage data rather than failing explicitly

3. **FILE* Lifespan**
   - `unz_s` holds a `FILE*` captured at `unzOpen()` time
   - If the underlying `.pk3` file is deleted/moved after `unzOpen()` but before `unzClose()`, subsequent I/O will silently fail with OS errors
   - No validation that file still exists when opening entries

4. **External Error Code Dependency**
   - `#define UNZ_ERRNO (Z_ERRNO)` ties this header to zlib.h being present and consistent
   - If zlib headers are unavailable or mismatched version, compilation or runtime behavior is undefined
