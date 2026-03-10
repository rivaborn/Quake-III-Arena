# code/jpeg-6/jcsample.c — Enhanced Analysis

## Architectural Role
This file is part of the vendored IJG libjpeg-6 library and implements the JPEG **encoder** downsampling pipeline—the inverse of what the runtime renderer needs. Although Quake III's renderer only **decodes** JPEG textures at runtime via `jload.c`, this compression module was vendored as part of the complete IJG library. It may be used by offline tools (level editor, asset pipelines) or remain dormant in the runtime engine. The file provides flexible per-component chroma subsampling, essential for the compression phase that client-side code never invokes.

## Key Cross-References

### Incoming (who depends on this file)
- **jcomapi.c, jcparam.c** (libjpeg): JPEG library initialization chain calls `jinit_downsampler` during compression context setup
- Offline tools (q3radiant, q3map, custom asset pipelines): *May* invoke JPEG encoding, though not confirmed in the provided cross-ref map
- **Runtime renderer**: Does **not** call this; only uses the JPEG *decoder* via `jload.c` → `jpg_load()` → decompression path

### Outgoing (what this file depends on)
- **jpeglib.h, jinclude.h**: Core IJG type definitions (`j_compress_ptr`, `jpeg_component_info`, `JSAMPARRAY`, etc.) and platform macros
- **jutils.c**: `jcopy_sample_rows()` for bulk row copying (called by `fullsize_downsample`)
- **Memory/Error subsystem** (`cinfo->mem->alloc_small`, `ERREXIT`, `TRACEMS`): Allocation and error handling via compression context callbacks
- **No cross-module dependencies within Quake III**; pure IJG standard library code

## Design Patterns & Rationale

**Polymorphic dispatch via function pointers:**  
`my_downsampler` holds per-component method pointers (`methods[MAX_COMPONENTS]`) selected at init time, enabling:
- **Specialization**: Dedicated fast paths for common 1:1, 2h1v, 2h2v ratios avoid generic integer-divide overhead
- **Conditional compilation**: Smoothing filter (`INPUT_SMOOTHING_SUPPORTED`) compiled conditionally, selected at runtime if enabled
- **Orthogonal configuration**: Each component can use a different sampler (e.g., Y channel 1:1, Cb/Cr 2h2v)

**Alternating-bias dithering:**  
`h2v1_downsample` and `h2v2_downsample` use alternating bias (0/1 or 1/2) when averaging pixels, preventing systematic rounding bias. This is a classic box-filter refinement: instead of always rounding 0.5 up (which would bias toward higher values), the code alternates, simulating an ordered dither pattern. This trades mathematical precision for perceptual quality—a tradeoff typical of image compression from that era.

**Fixed-point scaled arithmetic in smoothing:**  
`h2v2_smooth_downsample` and `fullsize_smooth_downsample` multiply smoothing weights by 2^16 to avoid floating-point, enabling integer-only math on embedded DSPs (crucial for 1994-era targets). Scales like `memberscale = 16384 - smoothing_factor * 80` derive from the formula `(1 - 5*SF)/4` where SF = smoothing_factor/1024.

## Data Flow Through This File

**Initialization (once per compression session):**
1. `jinit_downsampler()` called from the JPEG encoder init chain
2. Allocates `my_downsampler` subobject from the compression context's memory pool
3. Queries each component's sampling factors (`h_samp_factor`, `v_samp_factor`) and smoothing mode
4. Selects appropriate per-component downsampler and wires it into the public `jpeg_downsampler` vtable
5. If smoothing enabled and supported, sets `need_context_rows = TRUE` to request extra padding rows

**Per-row-group processing (many times during compression):**
1. `sep_downsample()` called for each "row group" (unit of vertical processing)
2. Dispatches to the selected per-component methods
3. Each method:
   - Calls `expand_right_edge()` to pad input to output width (avoiding boundary checks in inner loops)
   - Averages source pixels into output samples using the box-filter formula
   - Writes packed output to `output_buf`

**Example (2h2v case):** 2×2 input pixels → 1 output pixel, with bias alternation (1→2→1) to reduce rounding error.

## Learning Notes

**What this file illustrates:**
1. **Image resampling fundamentals**: Box filtering, bias reduction through ordered dithering, context padding for boundary conditions
2. **Polymorphic C design**: Function-pointer tables and subobject pattern were common before C++ became standard, avoiding vtable overhead and enabling compile-time specialization
3. **Fixed-point arithmetic**: Pre-FPU era optimization; multiply by 2^16 scales, compute integer, then shift right—still used in embedded graphics
4. **Era-specific tradeoff**: Smoothing via weighted neighbor sampling (not separable Gaussian) is computationally simpler than modern SSIM-aware compression

**Modern contrast:**
- Modern JPEG encoders (libjpeg-turbo) use SIMD vectorization and separate H/V passes for resampling
- Contemporary image codecs (HEIF, WebP) apply perceptually-aware filtering (e.g., guided filters) rather than box filters
- Quake III's actual runtime path never compresses—it decodes, so this code's design doesn't reflect runtime performance constraints

## Potential Issues

**Dead code risk:** If no Quake III tool or asset pipeline actually invokes JPEG encoding, `jcsample.c` may be entirely unused at both build and runtime, making it a maintenance burden. *Not immediately inferable from context whether offline tools use it.*

**Unsupported configurations:** `jinit_downsampler()` calls `ERREXIT(cinfo, JERR_CCIR601_NOTIMPL)` for CCIR 601 cosited sampling and silently disables smoothing (with trace message) for h2v1 and non-standard ratios. This matches 1990s JPEG encoder limitations but modern variants may expect these cases to work.

**Integer overflow risk (minor):** In `h2v2_smooth_downsample`, `neighsum += neighsum` doubles; if input max is 255, worst case is ~2040 before scaling. Scales fit in `INT32` but could be tighter. Not exploitable via image data alone.
