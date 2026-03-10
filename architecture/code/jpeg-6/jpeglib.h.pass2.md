# code/jpeg-6/jpeglib.h — Enhanced Analysis

## Architectural Role

This header sits at the **texture-load boundary** of the rendering pipeline. When `tr_image.c` (renderer) loads a texture from a `.pk3` file during level initialization, it calls through the ijpeg-6 public API defined here. The file's role is purely as an **integration point** — a stable contract between Quake III's image loader and the vendored JPEG library, insulating the renderer from internal ijpeg-6 implementation changes.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** — The sole runtime consumer. During `R_LoadImage` or `R_FindImageFile`, calls `jpeg_create_decompress` → `jpeg_read_header` → `jpeg_start_decompress` → `jpeg_read_scanlines` → `jpeg_destroy_decompress` to decode texture data sourced from the virtual filesystem.
- **Build-time consumers**: `code/jpeg-6/*.c` implementation files (jdapimin.c, jdmaster.c, etc.) include this header to implement their respective subsystem structs and function bodies.

### Outgoing (what this file depends on)
- **`code/jpeg-6/jconfig.h`** — Platform/compiler detection; defines feature flags like `HAVE_PROTOTYPES`, `JDCT_DEFAULT`, max block limits.
- **`code/jpeg-6/jmorecfg.h`** — Type abstractions (`JSAMPLE`, `JCOEF`, `UINT8`, `JDIMENSION`) that isolate ijpeg-6 from platform integer widths and memory models.
- **`code/jpeg-6/jpegint.h` / `code/jpeg-6/jerror.h`** — Conditionally included only when `JPEG_INTERNALS` is defined (i.e., by implementation files, not by tr_image.c).

## Design Patterns & Rationale

### Pluggable Manager Pattern
The file exposes three "manager" abstractions — **error**, **memory**, **source/destination**, **progress** — each as a struct of function pointers initialized at startup. This decouples ijpeg-6 from the engine's memory allocator, error handling, and I/O layer:
- Engine passes a `jpeg_error_mgr` pre-initialized by `jpeg_std_error()`, allowing ijpeg-6's `ERREXIT` macros to invoke engine error handlers.
- Memory is allocated/freed through `jpeg_memory_mgr` callbacks, which the engine can wire to `Hunk_Alloc` or zone pools.
- Input comes from a `jpeg_source_mgr` (typically `jpeg_stdio_src` or a custom buffer wrapper) — tr_image.c can provide a custom source that reads from the VFS.

**Rationale**: Quake III's "plugin" philosophy. By requiring managers to be plugged in, ijpeg-6 never hardcodes `malloc()` or `fopen()`, making it embeddable in any host (engine, tools, headless server).

### Master Struct + Common Fields Macro
`jpeg_common_fields` is a macro expanded into both `jpeg_compress_struct` and `jpeg_decompress_struct`. This avoids duplication of core fields (`err`, `mem`, `progress`, `is_decompressor`, `global_state`) while allowing each master struct to define type-specific fields below.

**Rationale**: C pre-C++ polymorphism. A generic `j_common_ptr` can safely upcast and access the first N fields; type-discriminated dispatch via `is_decompressor` flag enables shared utility code.

### Separate Path for Compression vs. Decompression
The struct hierarchy exposes both `jpeg_compress_struct` and `jpeg_decompress_struct` with minimal overlap. In Quake III's context, only decompression is used at **runtime**; compression would be reserved for **offline tools** (e.g., if the toolchain pre-encodes textures).

**Rationale**: Clean separation of concerns; applications that only need decoding can ignore compression fields and code paths.

## Data Flow Through This File

### Decompression (Runtime Texture Load)
```
tr_image.c: FS_ReadFile(texture.jpg) → buffer in RAM
       ↓
jpeg_create_decompress(cinfo)
jpeg_stdio_src(cinfo, FILE*) [or custom buffer source]
       ↓
jpeg_read_header(cinfo, TRUE)  [parses SOI, SOF, etc.]
       ↓
jpeg_start_decompress(cinfo)  [allocates output buffers, selects IDCT algorithm]
       ↓
Loop: jpeg_read_scanlines(cinfo, output, rec_outbuf_height)  [IDCT, color conversion]
       ↓
tr_image.c: resample, gamma-correct, upload to GPU
       ↓
jpeg_finish_decompress(cinfo) → jpeg_destroy_decompress(cinfo)
```

**Key state transitions**:
- `next_scanline` in cinfo tracks decompression progress; renderer polls this to drive its loop.
- Submodules (entropy decoder, inverse DCT, upsampler, color converter) are initialized lazily in `jpeg_start_decompress`, hidden behind opaque pointers in the struct.

## Learning Notes

### Library Integration Pattern
This file exemplifies **how to embed a complex C library into a larger engine**:
1. **Stable public API** (this header) remains unchanged across ijpeg-6 minor versions.
2. **Pluggable dependencies** (managers) instead of hardcoded I/O or memory.
3. **Versioning** (`JPEG_LIB_VERSION = 60`) allows graceful fallbacks if multiple versions are available.
4. **Dual compilation modes** (`JPEG_INTERNALS` gate) separate public client headers from internal implementation details.

### Idiomatic Choices for the Era (1990s)
- **Manual initialization order**: Unlike modern libraries with single `Init()` call, this requires:
  1. Allocate master struct on stack/heap
  2. Call `jpeg_std_error()` to wire error mgr
  3. Call `jpeg_create_decompressor()`
  4. Wire remaining managers
  5. Call `jpeg_read_header()`
  
  Modern APIs (libpng, libjpeg-turbo) reduce this boilerplate.

- **Explicit scanline access**: No abstract image handles. Caller directly manages the scanline loop; no iterator abstraction.

- **Constant proliferation**: `NUM_QUANT_TBLS`, `DCTSIZE`, `MAX_COMPS_IN_SCAN` are all hardcoded in headers rather than queried at runtime or in a capability struct. This reflects the immutability of the JPEG standard.

### Connections to Engine Architecture
The **pluggable manager pattern** mirrors Quake III's broader **vtable-based modularity**:
- Renderer exported as `refexport_t` DLL vtable.
- botlib exported as `botlib_export_t` function-pointer table.
- Game VM dispatches through `trap_*` syscall range.

All use the same philosophy: **invert dependencies** so the core engine never directly links to subsystems; instead, subsystems plug into the engine's service ports.

## Potential Issues

- **No issues inferable from this header alone**. However, in context:
  - ijpeg-6 is **vintage C (1990s)**: no explicit memory bounds, no buffer-overflow checks in the Huffman decoder. Modern builds should run under Address Sanitizer.
  - **Lossy format in a texture cache**: JPEG artifacts persist in memory; if the engine applies texture filtering on top, banding may become visible. A mipmap ladder compressed from the decoded scanlines will amplify quantization noise.
  - **Transcoding not used**: The file exposes `jpeg_read_coefficients`/`jpeg_write_coefficients` for lossless DCT manipulation (e.g., rotate without recompression). Quake III doesn't use this; all textures are fully decompressed and resampled, so any opportunity for lossless transformation is lost.
