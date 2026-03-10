# q3radiant/Bmp.cpp — Enhanced Analysis

## Architectural Role

This file implements a minimal BMP image loader/writer utility for Q3Radiant (the offline level editor). It is completely isolated from the runtime engine and provides only uncompressed 8-bit and 24-bit BMP support. It serves the editor's asset pipeline—specifically, texture preview and import—but has no bearing on shipped engine code or runtime dependencies.

## Key Cross-References

### Incoming (who depends on this file)
- Other Q3Radiant modules loading/saving textures during map editing (likely `TextureLoad.cpp`, `TexWnd.cpp`, or similar UI/preview systems)
- Dialog/import workflows that need to persist or display texture bitmaps
- These are **editor-only call sites**—never referenced by runtime engine modules

### Outgoing (what this file depends on)
- Standard C library: `<stdio.h>`, `<malloc.h>`, `<string.h>` for I/O and memory
- `bmp.h` header—defines `bitmap_t`, `bmphd_t`, `binfo_t`, `drgb_t` structures and constants (`BMP_SIGNATURE_WORD`, compression type flags `xBI_NONE`, `xBI_RLE8`, `xBI_RLE4`)
- Global `Error()` function (defined elsewhere in editor codebase)—non-recoverable error handler with `longjmp`-like semantics (never returns)

## Design Patterns & Rationale

**Utility/Service Module:** Encapsulates all BMP format parsing logic away from higher-level editor concerns (UI, selection, rendering).

**C-style Resource Management:** Uses `malloc`/`free` directly rather than C++ RAII. This was typical for late-1990s game tools; no exception-safe cleanup patterns.

**Format-Specific Helper Functions:**
- `BMPLineNone`, `BMPLineRLE8`, `BMPLineRLE4`: Dispatch based on compression type
- `BMPLine`: Router that delegates to appropriate decompression function
- Parallel encode/decode symmetry (`BMPLine` vs `BMPEncodeLine`)

**Explicit Data Transformation:** 
- BGR↔RGB byte-swap during I/O (lines 75–78, 238–241) to match internal RGB convention—necessary because BMP stores colors in BGR order.
- Row padding alignment to 4-byte boundaries (standard BMP requirement, lines 51–52, 229–230)

**Why this structure?** The code prioritizes simplicity: linear parsing, single-pass I/O, predictable memory layout. No streaming, no lazy loading, no compression beyond stubs. Appropriate for a level editor's texture preview pipeline.

## Data Flow Through This File

**LoadBMP flow:**
1. Open file, validate BMP signature word (`bfType != 0x4D42` → error)
2. Read `bmphd_t` header (14 bytes: file metadata)
3. Read `binfo_t` info header (40 bytes: image metadata)
4. Extract width, height, bpp; validate bpp ∈ {8, 24}
5. **If 8-bit:** Allocate and read 256-entry palette (4 bytes per DRGB entry)
6. Seek to pixel data start (`bhd.bfOffBits`)
7. Allocate pixel buffer: `width × height × pixbytes`
8. **Per scanline:**
   - Call `BMPLine()` to read raw line into scanline buffer (with padding stripped)
   - If 24-bit: swap BGR→RGB
   - Copy scanline into output buffer at reversed Y position (`biHeight - i - 1`) to flip image vertically
9. Free temporary scanline, close file, return populated `bitmap_t`

**WriteBMP flow:** (Reverse, with minor header computation)
1. Open file for writing
2. Write placeholder header (will update later)
3. Write info header with palette size 256 (even for 24-bit—minor bug)
4. **If 8-bit:** Write palette (256 × 4 bytes)
5. Record bitmap data offset
6. **Per scanline (reversed, top-to-bottom):**
   - Call `BMPEncodeLine()` to swap RGB→BGR and write with padding
7. Compute final file size
8. Seek back, update header with correct file size and data offset
9. Close file

**Transformations:**
- Input pixel data (in-memory RGB) → BMP file (BGR + padding)
- BMP file (BGR + padding) → in-memory RGB pixels
- Vertical flip during both read and write (BMP convention: bottom-up)

## Learning Notes

**Idiomatic to 1990s game tooling:**
- No `const` correctness (`char *fmt` vs `const char *fmt`)
- Bare pointer I/O with manual malloc/free (no STL containers, which were less mature/adopted then)
- Static helper functions (`GetColorCount`, `BMPLineNone`, etc.) to avoid namespace pollution
- `reinterpret_cast<>` used liberally, even for alignment-safe casts (`malloc` → `unsigned char*`)

**Format knowledge:**
- Understands BMP file structure intimately: signature word, twin headers (file + info), optional palette, stride padding
- Aware that BMP stores BGR and images are bottom-up (hence the vertical flip)
- RLE compression stubs exist but are not implemented—likely a "nice to have" never completed

**What modern engines do differently:**
- Load images into a unified asset pipeline (GPU texture formats, mips, compression)
- Support multiple formats (PNG, TARGA, DDS) via a pluggable codec system, not format-specific functions
- Use smart pointers and RAII for resource cleanup
- Stream large images rather than load entirely into RAM
- Validate image dimensions against hardware limits *before* allocation

**Connection to engine concepts:**
- Not directly related to ECS, scene graphs, or runtime engine patterns
- Pure data-format I/O utility—orthogonal to the game engine architecture shown in the overview
- Similar role to `tr_image.c` (in runtime renderer), but for *offline* (editor) use only

## Potential Issues

1. **Memory safety:** No validation of `biWidth` × `biHeight` before allocation. A malformed BMP with dimensions like `0x7FFFFFFF × 3` could cause `malloc` to fail silently or crash.

2. **Error handling:** No recovery mechanism. If `fread` fails mid-image, memory is allocated but not freed (no `fclose` in some error paths, lines 163, 182, 215—though line 194 does cleanup). Depends on external `Error()` to `longjmp` out, which leaks caller's stack frame.

3. **RLE stub functions (lines 92–100):** These unconditionally error. If a user loads a compressed BMP, the editor crashes with "RLE8 not yet supported" message. Should probably be silently converted or rejected earlier.

4. **Hard-coded palette size (line 305):** Writes `biClrUsed = 256` even for 24-bit images (which have no palette). Minor but semantically incorrect.

5. **Casting (lines 199, 210, 381):** `reinterpret_cast` used where `static_cast` or direct assignment would be more idiomatic/safe. Example: `malloc()` returns `void*`, which can implicitly cast to any pointer in C, so the explicit cast is unnecessary.

6. **No bounds checking on palette reads (lines 202–207):** Reads 256 entries but never validates that the file is long enough. If `fread` returns fewer bytes, stale memory is used.

7. **Platform I/O:** Uses C `stdio` directly rather than the engine's `FS_*` virtual filesystem, which is appropriate for editor tools but means texture paths are hardcoded (can't use `.pk3` search paths).
