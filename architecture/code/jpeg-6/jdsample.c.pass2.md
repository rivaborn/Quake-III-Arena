# code/jpeg-6/jdsample.c — Enhanced Analysis

## Architectural Role

This file implements the **upsampling stage** of the JPEG decompression pipeline, a critical step in the renderer's texture-loading pathway. When the renderer loads JPEG textures (via `code/renderer/tr_image.c` → `code/jpeg-6/jload.c`), each JPEG component may have been stored at a reduced sampling resolution (e.g., 2:1 horizontal subsampling for chroma). This file's responsibility is to expand those subsampled components back to full output resolution before color conversion. The module is initialized once per decompression via `jinit_upsampler` and then driven per-scanline-group by the decompressor's main pipeline in `jdmainct.c`. Because JPEG texture loading is synchronous and happens infrequently (once per asset), the code prioritizes correctness and modularity over extreme performance, though it still includes fast paths for common 2:1 cases.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer (`code/renderer/tr_image.c`)**  
  Indirectly: the renderer's texture loading calls `jload.c` (IJG entry point), which internally wires `jinit_upsampler` during decompression initialization.
- **jdmainct.c** (decompressor main controller)  
  Calls `sep_upsample` per-scanline-group to drive the upsampling pipeline.
- **jdpostct.c** (post-processing controller)  
  May cooperate with upsampler on row-group boundaries.

### Outgoing (what this file depends on)
- **jutils.c**  
  Calls `jcopy_sample_rows` (row duplication for vertical expansion) and `jround_up` (buffer sizing).
- **jpeglib.h / jpegint.h**  
  Reads/writes engine-visible structures: `jpeg_upsampler` (public vtable), `jpeg_component_info`, `jpeg_decompress_struct`, memory allocation callbacks.
- **Color converter** (`cinfo->cconvert->color_convert`)  
  Downstream consumer: `sep_upsample` fills `color_buf`, then immediately invokes the color converter to produce final RGB(A) output.

## Design Patterns & Rationale

**1. Strategy Pattern (Pluggable Upsampling Methods)**  
The `upsample1_ptr methods[]` array selects the optimal per-component upsampling strategy at init time based on sampling factors. This avoids branch-per-pixel overhead at runtime while keeping the pipeline flexible. Six distinct strategies are supported:
- `fullsize_upsample`: zero-copy passthrough for full-size components.
- `noop_upsample`: safety stub for unused components.
- `int_upsample`: generic integer-ratio box filter.
- `h2v1_upsample` / `h2v2_upsample`: fast box-filter specializations for 2:1 common case.
- `h2v1_fancy_upsample` / `h2v2_fancy_upsample`: higher-quality triangle filter using linear interpolation with alternating dither bias.

**2. Separation of Concerns (Upsampling + Color Conversion)**  
The intermediate `color_buf` decouples upsampling (per-component, format-agnostic) from color conversion (format-aware, full-resolution). This allows the upsampler to remain independent of the final color space (RGB, YCbCr, etc.).

**3. Lazy Buffer Allocation**  
Only components that actually require rescaling allocate intermediate buffers (`need_buffer` flag). Full-size components use zero-copy passthrough; unneeded components (set `component_needed = FALSE` by the app) skip allocation entirely. This minimizes memory footprint for common JPEG profiles.

**4. Fast Paths for Common Cases**  
The initialization logic special-cases 2:1 H and/or V ratios, which dominate real-world JPEG files (4:2:0 chroma subsampling). Within each case, it further chooses between simple box filtering (fast, artifacts visible) and fancy triangle filtering (slower, visually better). The choice respects `do_fancy_upsampling` and requires minimum component width for interpolation stability.

## Data Flow Through This File

```
Input (per row group):
  input_buf[ci][row] ← compressed component samples (subsampled resolution)
  in_row_group_ctr   ← tracks position in input row-group sequence

Processing (sep_upsample loop):
  1. When color_buf empty: invoke per-component methods[ci]
     - Transform subsampled input_buf[ci] → full-resolution color_buf[ci]
  2. Color convert: cconvert->color_convert(color_buf) → RGB(A) output
  3. Emit: advance out_row_ctr

Output (per scanline):
  output_buf[out_row_ctr + i] ← final RGB(A) pixels (full resolution)
  rows_to_go countdown ← guards against non-multiple image heights
```

**Buffer Lifecycle:**
- `next_row_out` tracks the current read position within `color_buf` (0 to `max_v_samp_factor`).
- When `next_row_out >= max_v_samp_factor`, the buffer is "empty"; the next call refills it from `input_buf[ci]`.
- The `rows_to_go` countdown ensures we don't emit more rows than the original image height (handles bottom-of-image padding).

## Learning Notes

**1. Modular Pipeline Design (1990s C Style)**  
This file demonstrates how to build a pluggable, stage-based image processing pipeline without C++ virtual methods. The vtable pattern (`upsample1_ptr`, `cinfo->cconvert->color_convert`) is the classical way to achieve polymorphism in procedural C, used consistently throughout the JPEG library and the broader Quake III engine (e.g., renderer vtables, botlib interfaces).

**2. Optimized Interpolation Without Lookup Tables**  
The "fancy" triangle-filter implementations (`h2v1_fancy_upsample`, `h2v2_fancy_upsample`) compute weighted sums directly using fixed-point arithmetic and bit-shifts. The clever **alternating dither bias** (biasing up on even pixels, down on odd) prevents systematic rounding error that would otherwise skew the image toward larger values. This is a micro-optimization from the pre-GPU era when every cycle counted.

**3. Era-Specific Assumptions**  
The code assumes:
- **No SIMD**: scalar loops over pixels; modern engines would use SSE/AVX for upsampling.
- **Row-wise I/O**: separates horizontal and vertical expansion; modern pipelines might vectorize both simultaneously.
- **Fixed small dimensions**: `MAX_COMPONENTS = 10`, `max_v_samp_factor ≤ 4`; scalable for desktop but not video.
- **No concurrency**: single-threaded texture loading; modern engines load textures on worker threads.

**4. Connection to Broader Engine Context**  
- This vendored libjpeg is the **only** JPEG decoder in the engine. All `.jpg` textures route through this upsampler.
- The renderer calls `RE_LoadImage` → `R_LoadJPG` (in `tr_image.c`) → libjpeg decompress pipeline.
- After upsampling and color conversion, the texture is gamma-corrected, mipmapped, and uploaded to VRAM by `tr_image.c`.
- Because JPEG loading is an offline asset-load operation (not per-frame), this code's latency is invisible to gameplay.

**5. Contrast with Compression Path**  
There is **no JPEG encoder** in the Quake III runtime; textures are pre-compressed offline. This asymmetry is typical: game engines decode many formats but rarely encode them.

## Potential Issues

- **No bounds checking on `input_data[inrow]` access** (e.g., in `int_upsample`, `h2v2_fancy_upsample`): assumes the caller provides valid row pointers. Malformed JPEG could cause out-of-bounds read, though the decompressor's integrity is the first line of defense.
- **Context-row assumption in fancy methods**: `h2v2_fancy_upsample` reads `input_data[inrow±1]` without explicit validation. The init code sets `need_context_rows = TRUE` to tell `jdmainct.c` to provide adjacent rows, but there's no assertion if context is missing.
- **Fixed `MAX_COMPONENTS` limit** (10): not a practical issue for JPEG (≤4 components) but reveals the library's age when it was designed for a broader set of formats.

---

**Data flow summary:** JPEG compressed samples (possibly subsampled) → **upsampling** → full-resolution intermediate buffer → **color conversion** → final RGB(A) output → renderer texture upload.
