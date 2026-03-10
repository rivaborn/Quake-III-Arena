# code/jpeg-6/jdcolor.c — Enhanced Analysis

## Architectural Role

This file implements the final **colorspace conversion stage** of JPEG decompression, sitting in the texture loading pipeline between the DCT decoder and the renderer's texture cache. When the renderer loads a JPEG texture via `TR_LoadImage` (in `code/renderer/tr_image.c`), the libjpeg library uses these functions to convert raw YCbCr component planes into the application's requested output format (RGB for framebuffer uploads, CMYK for print workflows, grayscale for simple textures). The module is critical for throughput: it processes decoded component data at the pixel loop's innermost level, so every cycle counts in a rendering engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer texture loader** (`code/renderer/tr_image.c`): Calls `jpeg_decompress_struct` machinery, which dispatches through `jinit_color_deconverter` during decompression initialization
- **libjpeg-6 post-processing pipeline**: This file is wired into the decompressor by `jdmaster.c` (another vendored module) at the end of the pipeline sequence
- No direct engine dependencies; all I/O goes through `cinfo->mem->alloc_small` (libjpeg's internal memory manager)

### Outgoing (what this file depends on)
- **libjpeg internals only**: `jinclude.h`, `jpeglib.h`, `jmorecfg.h` for macro definitions and the decompressor structure
- **Memory allocator**: `cinfo->mem->alloc_small` for lookup table allocation (allocated once per decompression context, not per frame)
- **Range-limiting table** (`cinfo->sample_range_limit`): Pre-populated by `jpeg_start_decompress` to clamp YCC→RGB arithmetic noise
- **jutils.c** (`jcopy_sample_rows`): For efficient grayscale passthrough

## Design Patterns & Rationale

**Precalculation for inner-loop performance**: The `build_ycc_rgb_table` function trades 4 × 256 bytes of memory for elimination of four multiplications per pixel in the hot path. This is a classic 1990s optimization: lookup tables were cheaper than multiply units on pentium-era CPUs. Modern engines use SIMD vectorization instead, but this design reflects the era when Quake III was developed.

**Fixed-point arithmetic** (`SCALEBITS=16`): Avoids floating-point math entirely. The conversion coefficients (1.40200, 0.71414, etc.) are pre-scaled by 2^16, and results are right-shifted to recover the final value. This is deterministic across platforms and faster than `float`-based math on embedded or older hardware.

**Template-method dispatch**: `jinit_color_deconverter` validates the JPEG color space and wires up the appropriate conversion function (`ycc_rgb_convert`, `grayscale_convert`, `null_convert`, etc.) through a function pointer. This is the classic way to implement polymorphism in C.

**Asymmetric G-channel tables**: The Cb and Cr contributions to green are pre-added (left scaled) and combined in the converter loop *before* rounding, rather than rounding each component separately. This is a subtle fixed-point optimization: `ONE_HALF` is baked into `Cb_g_tab[i]` to absorb the rounding in one final right-shift.

## Data Flow Through This File

1. **Initialization** (`jinit_color_deconverter`): Decompressor startup validates component counts, allocates color conversion state, and routes to `build_ycc_rgb_table` if YCC→RGB is requested.

2. **Table construction** (`build_ycc_rgb_table`): For each possible Cb/Cr sample value (0–255), precomputes the R, G, B deltas as scaled integers. The G deltas are left unrounded (scaled by 2^16) because both Cb and Cr contributions must be summed before quantizing.

3. **Per-row conversion** (`ycc_rgb_convert`, `ycck_cmyk_convert`, `null_convert`, `grayscale_convert`): The post-processing controller invokes the wired function for each row group. YCC converters read three planar input buffers (Y, Cb, Cr), look up per-sample deltas, apply range-limiting via `sample_range_limit[y + delta]`, and write interleaved RGB/CMYK output.

4. **Output**: Interleaved pixel data flows to the texture upload path in `tr_image.c`.

## Learning Notes

- **Register-heavy inner loops**: Notice the `register` keyword for loop variables and table pointers. Modern compilers ignore this, but it documents the original intent: keep hot data in CPU registers.
- **Why range-limiting is mandatory**: DCT quantization + chroma subsampling can push Y+Cr values outside [0, 255]. The `sample_range_limit` table (a 1024-entry LUT offset to handle both negative and overrange indices) clamps silently. This is idiomatic to JPEG decoders of the era.
- **YCCK→CMYK inversion**: The code computes CMY as `1 - RGB`, which is mathematically sound (inverted RGB is the same as CMY in the color cube). Passing K unchanged is the Adobe convention.
- **Contrast with modern engines**: Modern texture pipelines decompress to sRGB linear on the GPU or use compute shaders. This file represents CPU-side decompression, blocking the main thread—a bottleneck that modern engines avoid via async decode and streaming.

## Potential Issues

- **Table allocation waste**: `build_ycc_rgb_table` allocates four 256-entry tables per decompressor instance. If many JPEGs are decoded in parallel (unlikely on single-threaded Q3A but possible on modern systems), this multiplies memory footprint. A global cached lookup table would be more efficient.
- **Fixed-point overflow risk** (low likelihood): If `SCALEBITS` is accidentally changed to 17 or higher by an editor, the math silently produces wrong results (no saturating arithmetic). The fixed-point constants are tightly coupled to this macro.
- **No graceful degradation**: If `cinfo->sample_range_limit` is NULL (initialization bug upstream), the code crashes. No null-check.
