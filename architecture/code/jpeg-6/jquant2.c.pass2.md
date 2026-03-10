# code/jpeg-6/jquant2.c — Enhanced Analysis

## Architectural Role

This file implements Heckbert-style 2-pass color quantization, part of the vendored IJG libjpeg-6 library. In the Q3A engine, it serves texture asset loading: when the renderer (`tr_image.c`) decompresses JPEG images during load-time, this module reduces full-depth RGB to a custom palette (typically 8-bit indexed, though Q3A's final textures use direct RGB). The quantizer runs offline (per-texture load), not per-frame, making performance secondary to quality.

## Key Cross-References

### Incoming (who depends on this)
- **code/renderer/tr_image.c** — calls JPG decompression pipeline during `R_LoadImage`. Invoked once per unique texture asset.
- **code/renderer/tr_init.c** — indirectly, during renderer initialization when default textures are loaded.
- Engine is texture-load-time consumer only; no per-frame dependency.

### Outgoing (what this file depends on)
- **jpeglib.h / jinclude.h** — JPEG infrastructure: memory allocation (`alloc_small`, `alloc_large`, `alloc_sarray`), error handling (`ERREXIT`), decompressor state (`j_decompress_ptr`).
- **jmorecfg.h** — component ordering macros (`RGB_RED`, `RGB_GREEN`, `RGB_BLUE`), JSAMPLE precision (`BITS_IN_JSAMPLE`).
- No Q3A engine subsystem dependencies (self-contained within libjpeg).

## Design Patterns & Rationale

### Heckbert Median-Cut Algorithm
The file implements the seminal 1982 approach: partition the input color space iteratively by splitting the "largest" box (by population, then volume) along the longest scaled axis. This greedy strategy produces perceptually-balanced palettes without expensive tree-based optimization.

**Tradeoff**: Median-cut is O(n log n) in palette size with simple implementation, vs. more sophisticated clustering (k-means, octree) that demand faster hardware or offline preprocessing.

### Memory Segmentation (DOS Era Legacy)
The 3D histogram uses a three-level indirection pointer hierarchy (`hist3d` → `hist2d[]` → `hist1d[]`) rather than a flat array. This fits within DOS/16-bit machine near/far memory segments. The `UINT16` cell type saves 128 KB vs. 256 KB for full-precision counts.

**Rationale**: Written for 1990s systems with ~640 KB conventional memory. Modern code would allocate flat, use 32-bit counters, and not worry.

### Overflow Saturation Pattern
Histogram cells saturate: `if (++(*histp) <= 0) (*histp)--` detects wrap-around and undoes it. This "clamping without clamping" avoids explicit `if` after every increment.

**Rationale**: Avoids O(histogram_size) initialization and conditional post-increment overhead. Overflow is rare with JPEG's limited pixel count per cell.

### Lazy Inverse-Map Cache
The histogram is repurposed as an inverse lookup table after quantization. `fill_inverse_cmap` fills subboxes on-demand, avoiding O(palette_size) nearest-color scans.

**Rationale**: Trades one allocation reuse for moderate per-pixel cache-miss overhead. Acceptable for offline load-time quantization.

## Data Flow Through This File

```
Input: RGB image bytes (pass 1)
   ↓ prescan_quantize
Histogram: 3D cell grid, one entry per quantized (R,G,B) tuple
   ↓ finish_pass1 → select_colors
Box List: iteratively split via median_cut until palette size reached
   ↓ compute_color (per box)
Colormap: representative RGB color per palette index, stored in cinfo->colormap
   ↓ start_pass_2_quant
Inverse Map Cache: histogram reused, lazily filled with palette indices
   ↓ pass2_no_dither or pass2_fs_dither
Output: 8-bit palette indices (or dithered indices if F-S enabled)
```

**State transitions**: `prescan_quantize` → `finish_pass1` zeroes F-S buffers if needed → `pass2_*_dither` consumes the colormap.

## Learning Notes

### Idiomatic to JPEG Era (1990s)
- **Segmented memory model**: Assume 64 KB segment limits; indirection trees are necessity, not design pattern.
- **Histogram saturation**: UINT16 cells trade accuracy for memory; overflow is acceptable clamping.
- **Error dithering table**: Pre-computed LUT avoids per-pixel conditional branching (cache-unfriendly on 486/Pentium).

### Contrast with Modern Graphics
- **GPU-accelerated**: Texture quantization would run as compute shader; palette selection could use GPU-parallel k-means.
- **No indexed color**: Modern hardware (DXT, BC, ASTC) uses block-based compression; palettes are anachronistic.
- **Direct linear allocation**: Flat arrays, 32-bit counts, no far pointers.

### Computer Graphics Concepts
- **Perceptually-weighted distance**: Uses NTSC luma weights (R:G:B = 2:3:1) to bias cuts toward green, which the eye is most sensitive to.
- **2-norm volume**: Uses Euclidean distance rather than true volume to penalize narrow boxes; a box is splittable iff `volume > 0`.
- **Floyd-Steinberg: error diffusion**: Classic ordered-dithering alternative, spreading quantization error to unprocessed neighbors with fixed weights (7/16, 3/16, 5/16, 1/16).

## Potential Issues

1. **Integer Precision in Volume Calculation** — The 2-norm uses left-shifts and squared components: `dist0*dist0 + dist1*dist1 + dist2*dist2`. For large boxes (high histogram precision), intermediate products could overflow INT32. Mitigated in practice by histogram compression (5–6 bits).

2. **Histogram Reuse as Inverse Map** — Overwriting histogram with colormap indices means pass 1 data is lost. If rerunning quantization with different parameters requires a fresh histogram, must call `start_pass_2_quant(cinfo, TRUE)` to re-zero. Not an issue in single-quantization workflows.

3. **Floyd-Steinberg Error Buffer Size** — Allocated as `(width + 2) * 3 * sizeof(FSERROR)` per row. For very wide images, this could exceed near-segment limits; code does allocate with `alloc_large` (far memory), but the trade-off is slower access.

4. **No Adaptive Bit Depth** — Histogram precision (5–6 bits) is compile-time constant. High-depth images (e.g., 16-bit input) quantize coarsely; lower-depth images (e.g., 4-bit input) over-allocate histogram space.

---

**Summary**: `jquant2.c` is a self-contained, well-optimized color quantization module exemplifying 1990s graphics programming. Its role in Q3A is texture asset loading only; the engine itself works in full RGB space. The Heckbert algorithm and Floyd-Steinberg dithering are timeless techniques, but memory management and integer tricks reflect era-specific constraints no longer relevant to modern engines.
