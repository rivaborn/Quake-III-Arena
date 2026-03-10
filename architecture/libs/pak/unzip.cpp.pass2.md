# libs/pak/unzip.cpp — Enhanced Analysis

## Architectural Role
This file provides the low-level ZIP decompression machinery that underpins Quake III's virtual filesystem. The engine's transparent mounting of `.pk3` files (ZIP archives) alongside physical directories depends on this layer to decompress individual entries on-the-fly. While residing in `libs/` (tool-shared headers), its twin in `code/qcommon/unzip.c` is integrated into the runtime engine's `files.c` virtual filesystem (`FS_*` API), making ZIP support a fundamental architectural capability.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/qcommon/files.c`** and the broader FS subsystem: invoke unzip functions to open `.pk3` files and extract individual entries during `FS_FOpenFileRead` / `FS_ReadFile` with transparent ZIP mounting
- **`code/qcommon/cmd.c`** (indirectly via FS): asset loading for every subsystem funnels through FS, which may decompress from ZIP
- **Off-line tools** (`q3map`, `bspc`, `q3asm`): likely reuse this same library via `libs/pak/` stubs for multi-format asset loading without needing the full engine
- **Client-side UI/cgame**: asset streaming (textures, models, scripts) resolved through virtual FS, transparently decompressed

### Outgoing (what this file depends on)
- **System file I/O**: `fopen`, `fseek`, `fread`, `fclose` for raw `.pk3` file access
- **zlib decompression**: `inflate()` call (declared but implementation elided in truncated view) for stream decompression
- **CRC checksums**: adler32 validation for data integrity (defined in header)
- **No engine callbacks**: entirely self-contained; does not call back into engine except through initial file handle creation

## Design Patterns & Rationale

**Embedded Library Pattern**: Rather than link against external zlib, the unzip layer is a thin, self-contained shim that bundles zlib's decompression constants and function signatures. This minimizes external dependencies for tools and ensures version consistency. The trade-off is code size and the risk of divergence if zlib is updated elsewhere.

**ZIP Format Abstraction**: Presents a simple sequential interface (find entry, read bytes, extract CRC) that hides the complexity of ZIP local file headers, central directory navigation, and variable-length field encoding. This is the key design win—callers in the FS layer don't manipulate ZIP structures directly.

**No Encryption**: By design, Q3's `.pk3` files are unencrypted; the code explicitly does not support PKZip 1.x encryption. This simplifies the library and reflects Quake III's philosophy of open, distributable pak files (as opposed to secrets/security).

## Data Flow Through This File

1. **Caller** (FS subsystem) requests entry from a `.pk3` file by name
2. **unzip library** navigates the ZIP's central directory to locate the entry's local header
3. **Compressed data** is streamed through `inflate()` (zlib)
4. **Decompressed bytes** returned to caller's buffer
5. **CRC validation** (adler32) confirms no corruption during extraction

The key insight: the FS layer (in qcommon) wraps this unzip API into a unified handle system that makes `.pk3` files indistinguishable from directories during `FS_ReadFile` calls.

## Learning Notes

- **Era-specific design**: Early 2000s engines commonly embedded third-party code (zlib, libjpeg, etc.) rather than relying on system libraries. This file exemplifies that pattern.
- **Why ZIP for games**: PKWare's ZIP format was industry-standard for distributable game assets; simple, widely-supported, and compressed. Q3's `.pk3` is just a ZIP with a different extension.
- **Single-threaded I/O model**: No internal buffering or async decompression; each `FS_ReadFile` call decompresses synchronously. This is fine for interactive loading but naive by modern standards (no mmap, no prefetch).
- **Not an ECS or scene-graph concern**: This is pure I/O plumbing—invisible to gameplay logic. The abstraction barrier between FS and game code is what enables transparent ZIP mounting.

## Potential Issues

- **Zlib version lock**: The bundled zlib definitions (version 1.1.3, July 1998) are frozen in time. If the codebase's zlib implementation elsewhere evolves, this header may become inconsistent.
- **No async support**: All decompression is synchronous, blocking the frame. Large `.pk3` files or slow decompression could cause hitches (especially on late-1990s hardware). Modern engines use threaded I/O.
- **No streaming iterator**: The API requires knowing entry names upfront; there's no efficient way to list all files in a `.pk3` at runtime without parsing the entire central directory repeatedly.
- **Memory allocation**: Likely delegates allocation to the caller; no internal memory pool for long-lived inflate state, so each `FS_ReadFile` allocates and frees transient decompression buffers.

---

**Cross-architectural insight**: This file is a **seam** between the engine's public virtual FS API (`FS_ReadFile`, which clients call) and the physical storage layer (ZIP files on disk). Its simplicity is its strength—it does one thing (decompress ZIP entries) and does it without coupling to the broader engine, making it portable to tools and platforms alike.
