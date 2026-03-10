# code/jpeg-6/jquant1.c — Enhanced Analysis

## Architectural Role

This file implements the quantization stage of the JPEG decompression pipeline, positioned between color space conversion (`jdcolor.c`) and final output. It's part of the vendored **libjpeg-6** library, which the renderer loads exclusively at texture load time (`code/renderer/tr_image.c` → `jload.c`). Since Q3A targets real-time performance, the 1-pass quantizer prioritizes speed over quality—critical during map/asset streaming on low-bandwidth networks or slow I/O hardware typical of late-1990s consoles and PCs.

## Key Cross-References

### Incoming (who calls this module)
- **Renderer texture loading** (`code/renderer/tr_image.c`): Calls `jload.c` to decompress JPEG assets into the global image cache during renderer init and dynamic load requests.
- **libjpeg post-processing pipeline**: `jdpostproc.c` calls the installed `color_quantize` method pointer from the `jpeg_color_quantizer` vtable for each row batch.

### Outgoing (what this file depends on)
- **libjpeg memory manager** (`jmemmgr.c`): All allocations via `alloc_small`, `alloc_large`, `alloc_sarray` from `cinfo->mem`.
- **libjpeg error handling**: `ERREXIT1`, `TRACEMS*` macros for fatal and informational reporting.
- **Platform-neutral JPEG types**: `JSAMPLE`, `JDIMENSION`, `INT16/INT32`, `FAR` pointer syntax for far-heap allocation on 16-bit systems.

## Design Patterns & Rationale

**Orthogonal colormap (divide-and-conquer):**  
Each color component's quantization is independent; the N-dimensional colormap is constructed as a Cartesian product of per-component value sets. This allows `select_ncolors` to distribute colors greedily (starting from 2^N, incrementing G→R→B in RGB mode) rather than solving an intractable global optimization. The final colorindex lookup compounds per-component indices via premultiplied strides—a 1990s cache-line optimization avoiding runtime divisions.

**Padding for dithering boundary conditions:**  
Ordered dithering can produce pixel values outside the input range after adding dither noise. Rather than branch on every pixel to clamp, the colorindex table is padded at both ends (when `JDITHER_ORDERED` is active), allowing out-of-bounds indexing to gracefully map to edge values. This eliminates a per-pixel conditional in the hot path.

**Static Bayer matrix scaling:**  
The `base_dither_matrix` is a compile-time constant (generated via Stephen Hawley's algorithm). Per-component matrices are scaled once during init via `make_odither_array`, accounting for the actual number of output levels. This avoids runtime scaling in the quantize loop.

**Floyd-Steinberg error row buffering:**  
Errors are stored in a single row-sized buffer (plus padding), reused for both the current row (pending pixels) and the next row (already-processed pixels). This is a space-time tradeoff: store one row of errors instead of recomputing, at the cost of scan-direction alternation (even rows left-to-right, odd right-to-left).

## Data Flow Through This File

1. **Initialization** (`jinit_1pass_quantizer`):
   - Allocate `my_cquantizer` state struct
   - Call `create_colormap`: distribute colors, fill with equally-spaced values
   - Call `create_colorindex`: build premultiplied lookup tables (optionally padded for dithering)
   - Optionally preallocate Floyd-Steinberg error buffer

2. **Per-pass setup** (`start_pass_1_quant`):
   - Install the appropriate quantize method pointer (no-dither, ordered, or F-S) based on `cinfo->dither_mode`
   - If dithering mode changed, rebuild dither tables or FS workspace

3. **Per-row quantization** (one of `color_quantize`, `color_quantize3`, `quantize_ord_dither`, `quantize_fs_dither`):
   - Input: `JSAMPARRAY` rows from upstream (full RGB/YCbCr values, 8-bit per component)
   - Lookup: index into `colorindex[component][pixelvalue]` → stride offset, sum across components
   - Dither (optional): add Bayer matrix cell or F-S error value before lookup
   - Output: `JSAMPARRAY` colormap indices (8-bit per pixel)

## Learning Notes

**Idiomatic to 1990s quantization:**
This implementation reflects pre-GPU era constraints: no real-time 3D color space analysis, no adaptive palette per image, no perceptual color metrics. The equally-spaced colormap is mathematically simple and fast to construct, but suboptimal for photographic content. Modern engines (post-2000s) would either avoid quantization entirely (32-bit textures) or use octree/k-means clustering.

**Gamma correction gap:**
The code explicitly notes that representative values *should* be equidistant in linear (gamma-corrected) space to match human perception, but `jdcolor.c` doesn't apply gamma at the time of writing. This is a missed opportunity that could have improved visual quality at zero cost (just a nonlinear mapping in `output_value`).

**Dithering trade-offs:**
- **No dither**: 3D colormap banding visible; fastest.
- **Ordered dither**: Fixed pattern visible on flat areas; moderate cost (matrix lookup per pixel).
- **Floyd-Steinberg**: Best visual quality; highest cost (2–3 additions + error buffering per pixel).

The conditional compilation (`QUANT_1PASS_SUPPORTED`) suggests this was optional even in baseline libjpeg, likely for embedded or resource-constrained builds.

## Potential Issues

1. **Assumption: `ODITHER_SIZE` is power of 2** (line ~70): The `ODITHER_MASK` will silently misbehave if someone changes `ODITHER_SIZE` to a non-power-of-2. No runtime assertion.

2. **FS workspace allocation at high resolution**: Very wide images with F-S dithering allocate `(width + 2) * sizeof(FSERROR) * out_color_components` per component. On 16-bit builds (`FSERROR = INT16`), a 2000-pixel image with 3 components consumes ~12 KB per component—could exceed near-data segment on some platforms.

3. **No validation of colormap orthogonality**: Code assumes `create_colormap` produces orthogonal results. If a caller manually injects a non-orthogonal colormap, `create_colorindex` and quantization will produce incorrect indices without error.

4. **Endian-sensitive Bayer matrix**: The matrix is byte-order-agnostic (all values fit in 0–255 signed range), but multi-component dither scaling does `INT32` arithmetic with platform-dependent signedness behavior when rounding negative intermediate values (lines ~365–371).
