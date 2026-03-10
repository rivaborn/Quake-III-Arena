# libs/jpeg6/jpgload.cpp — Enhanced Analysis

## Architectural Role

This file provides a single-entry-point JPEG decompressor (`LoadJPGBuff`) bridging the renderer's texture loader to libjpeg-6. It consumes raw JPEG buffer data and outputs linear RGB(A) scanlines in host-endian format. Within the renderer's asset pipeline (`tr_image.c`), this sits between the filesystem layer (which reads `.jpg` files into memory) and downstream gamma-correction, mip-chaining, and GPU upload. The function is part of **code/jpeg-6**, a vendored third-party codec library with no external engine dependencies beyond memory allocation.

## Key Cross-References

### Incoming (who depends on this file)
- **code/renderer/tr_image.c** (`R_LoadImage`, texture image loading) – calls `LoadJPGBuff` when a `.jpg` extension is encountered
- **Renderer texture cache** – invokes during `R_CreateImage` to materialize JPEG assets into the runtime texture registry

### Outgoing (what this file depends on)
- **code/jpeg-6/jpeglib.h** – entire libjpeg public API (`jpeg_decompress_struct`, `jpeg_std_error`, `jpeg_create_decompress`, `jpeg_read_header`, `jpeg_read_scanlines`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`)
- **stdlib.h** (`malloc`, `memset`) – heap allocation for output buffer
- **Implicit qcommon memory subsystem** – assumes `malloc` is available (no custom allocator wrapping)

## Design Patterns & Rationale

**Single-Function Wrapper Pattern:** The file exports exactly one symbol, making it a thin facade over libjpeg's full decompression API. This isolates texture loading from codec complexity; the renderer never directly handles JPEG state machines.

**Deferred Allocation:** Output buffer is allocated once dimensions are known (after `jpeg_read_header`), avoiding two-pass reads. Pointer-to-pointer output parameter (`unsigned char **pic`) allows the allocator to return ownership to the caller without a struct wrapper.

**Stateless Design:** Each call is independent; no retained JPEG library state across frames. This simplifies error recovery and fits the renderer's stateless per-asset-load model.

**Implicit Format Conversion:** The code assumes libjpeg will output in one of its default color spaces (likely `JCS_RGB` or `JCS_EXT_RGBA`) and then **unconditionally overwrites alpha to 255**, padding all output to 4 bytes-per-pixel (RGBA). This is idiomatic for game texture pipelines (many shaders expect RGBA) but bakes the assumption into the loader.

## Data Flow Through This File

1. **Input:** Raw JPEG-compressed byte buffer (from disk, loaded by `FS_ReadFile`)
2. **Parse header** → Extract width, height, color space from JPEG metadata
3. **Allocate:** `malloc(width * height * components + 1)` for raw pixel data
4. **Decompress:** Read scanline-by-scanline into buffer via `jpeg_read_scanlines`
5. **Post-process:** Loop over all pixels, set `buf[i+3] = 255` (alpha channel)
6. **Output:** Caller receives ownership of heap buffer; owns lifetime responsibility thereafter
7. **Cleanup:** libjpeg state freed; caller must later free pixel buffer (typically during texture eviction or engine shutdown)

The **critical transition point** is the buffer pointer: `pic` is written once, caller assumes full ownership and must `free()` it when the texture is deleted.

## Learning Notes

**Idiomatic Q3A Texture Loading:** This exemplifies how Q3A decouples asset codecs from the renderer. The same pattern exists for `.tga` and other formats in `tr_image.c`'s switch statement. Modern engines often consolidate these into plugin-based decoders or unified image libraries (e.g., stb_image, libimagequant).

**Era-Specific Assumptions:**
- **No color space negotiation:** Assumes libjpeg's defaults; no ICC profile or CMYK handling
- **No streaming:** Entire decompressed image held in RAM; streaming/tiling would require refactoring
- **Unconditional RGBA:** Many Q3A shaders assume alpha channel, even for fully opaque textures; this loader pads accordingly

**Memory Ownership Hazard:** The returned buffer's lifetime is implicit—no RAII, no reference counting. Caller must manually `free()` or leak. Early error returns (e.g., `jpeg_read_header` failure) would leave caller with uninitialized pointers, requiring defensive null-checks in `tr_image.c`.

## Potential Issues

1. **Dangling Stack Pointer:** Line ~98–99 creates a pointer-to-pointer from a local variable:
   ```cpp
   bbuf = ((out+(row_stride*cinfo.output_scanline)));
   buffer = &bbuf;  // ← buffer now points to stack-local bbuf
   jpeg_read_scanlines(&cinfo, buffer, 1);
   ```
   While this *works* because `jpeg_read_scanlines` only reads the pointer within that call, it's fragile and non-idiomatic. Should be:
   ```cpp
   buffer = &out[row_stride * cinfo.output_scanline];
   ```

2. **No Error Handling:** `jpeg_read_header`, `jpeg_start_decompress`, and `jpeg_read_scanlines` return status codes that are ignored. If the buffer is truncated or malformed, the function will return with incomplete/garbage data rather than failing cleanly.

3. **Uninitialized Output on Error:** If any JPEG library call fails, `*pic`, `*width`, and `*height` may be partially written, leaving caller with dangling/invalid pointers.

4. **Hardcoded Alpha Assumption:** The loop `for ( i = 3 ; i < j ; i+=4 )` assumes 4-byte-per-pixel output. If libjpeg's `output_components` differs, this writes out-of-bounds.

5. **Off-by-One Allocation:** `malloc(nSize+1)` allocates one extra byte but never uses it; suggests copy-paste from null-terminated string code.
