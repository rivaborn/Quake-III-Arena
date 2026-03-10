# libs/jpeg6/jdsample.cpp — Enhanced Analysis

## Architectural Role
This module is part of the vendored **IJG libjpeg-6** library, specifically the JPEG decompression upsampling stage. In the Quake III Arena engine, it exclusively serves the **renderer's texture loading pipeline** (`code/renderer/tr_image.c`). When a `.jpg` texture is loaded, decompressed JPEG data with downsampled chroma components (typically 2:1 or 4:1 subsampling in horizontal/vertical dimensions) is upsampled back to full resolution before GPU upload via `R_LoadJPG`. This module's method selection and quality modes are selected *at texture load time*—a one-time cost, not a per-frame operation.

## Key Cross-References

### Incoming (who calls this)
- **`code/renderer/tr_image.c` (`R_LoadJPG`)**  — Invokes libjpeg decompression pipeline; `jinit_upsampler` is called by the libjpeg decompressor during `jpeg_read_header` → `jpeg_start_decompress` sequence
- **`code/jpeg-6/jdmainct.c`** — Main decompression controller; calls the upsampler's `sep_upsample` method once per row group during `process_data_ccontroller` loop
- **`code/renderer/tr_init.c` / `tr_main.c`** — Indirectly trigger texture loads and thus the JPEG pipeline during level initialization and dynamic texture binding

### Outgoing (what this module depends on)
- **libjpeg internal memory pool** — Calls `cinfo->mem->alloc_small` / `alloc_sarray` for color conversion buffers (backed by engine's `qcommon` Hunk allocator via `refimport_t` in renderer DLL)
- **`jdmainct.c` (`jcopy_sample_rows`)**  — Utility for row duplication in vertical expansion paths
- **Color converter vtable** (`cinfo->cconvert->color_convert`) — Post-upsample YCbCr→RGB or other color-space transformation; filled by `jcmainct.c`
- **`j_decompress_ptr cinfo` state** — Reads sampling factors, output dimensions, component metadata populated by JPEG header parsing

## Design Patterns & Rationale

**Strategy Pattern (Per-Component Method Selection)**  
`jinit_upsampler` analyzes each JPEG component's sampling ratio and selects an optimized method:
- **Full-size** → `fullsize_upsample` (zero-copy alias to input)
- **2:1 horiz only** → `h2v1_upsample` (simple box) or `h2v1_fancy_upsample` (linear interpolation)
- **2:1 both** → `h2v2_upsample` or `h2v2_fancy_upsample` (triangle filter, may request extra input row context)
- **Generic integral factors** → `int_upsample` with precomputed expansion factors
- **Unused/discarded** → `noop_upsample` (safety stub)

This avoids runtime conditionals in tight per-row loops.

**Quality/Speed Tradeoff**  
The "fancy" methods use weighted interpolation (triangle filtering) instead of naive pixel replication. Since JPEG decompression happens *once at texture load time*, not per frame, the added cost is invisible to gameplay. The `do_fancy_upsampling` cvar gates this based on `min_DCT_scaled_size > 1` (disables fancy on small thumbnails or when full DCT scaling is unavailable).

**Stateful Conversion Buffer**  
`color_buf[]` holds one upsampled row group until color conversion + client fetch consume it, avoiding redundant copy passes.

## Data Flow Through This File

1. **Input**: Downsampled JPEG component data from IDCT, organized in "row groups" (typically 8 or 16 rows of Y, 4 or 8 rows of CbCr)
2. **Processing**:
   - `start_pass_upsample`: Initialize row counters and mark buffer empty
   - `sep_upsample` (called per row group): Invoke component-specific upsampler → populate `color_buf[]` → feed to color converter → output to final RGBA buffer
3. **Output**: Full-resolution, color-converted sample rows ready for GPU texture upload

Row group size and expansion factors are precomputed at init time in `rowgroup_height[]` and `h_expand[] / v_expand[]` to minimize per-call overhead.

## Learning Notes

- **Legacy IJG Code**: This is unmodified public-domain JPEG library code circa 1994, embedded verbatim. No Quake-specific modifications visible; represents how vendors integrated third-party image codecs in mid-2000s engines.
- **Register-Intensive Loops**: Extensive use of `register` keyword and local variable hoisting (e.g., `thiscolsum, lastcolsum, nextcolsum` in `h2v2_fancy_upsample`) reflects Pentium-era compiler expectations. Modern compilers ignore these hints.
- **"Box Filter" Transparency**: Comments acknowledge that simple replication upsampling introduces artifacts; the fancy path with triangle filtering is the recommended production choice.
- **DSP-Like Structure**: Tight inner loops with pixel-at-a-time processing and hand-optimized fractional arithmetic (`>> 2`, `>> 4` for fixed-point division) are characteristic of image processing code before SIMD/GPU ubiquity.
- **Modern Contrast**: Unlike the engine's native code (which uses virtual method dispatch and late binding), this module uses static function pointers and compile-time tuning—typical of pre-OOP C library design.

## Potential Issues

- **Context Row Dependency**: `h2v2_fancy_upsample` requires access to adjacent input rows (`inrow-1`, `inrow+1`). The init code sets `upsample->pub.need_context_rows = TRUE` to signal the main decompressor to buffer extra rows, but if this flag is ignored upstream, buffer overruns could occur. *(Mitigated in practice by careful jdmainct.c integration.)*
- **Integer Overflow**: Fixed-point math in `h2v2_fancy_upsample` (e.g., `thiscolsum * 3 + nextcolsum + 7`) could overflow if intermediate sums exceed 32-bit range for 16-bit JSAMPLE. Code conditionally uses `INT32` when `BITS_IN_JSAMPLE != 8`, but 16-bit JPEG support is rare in practice.
