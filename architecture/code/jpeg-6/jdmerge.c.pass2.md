# code/jpeg-6/jdmerge.c — Enhanced Analysis

## Architectural Role
This file provides a high-performance optimization path for JPEG decompression when chroma is subsampled at 2:1 horizontal and 1:1 or 2:1 vertical ratios (2h1v / 2h2v, the most common cases in practice). It lives entirely within the vendored `code/jpeg-6/` JPEG library and is invoked indirectly during texture asset loading in `code/renderer/tr_image.c` via the public libjpeg API. The module is optional—controlled by compile-time `#ifdef UPSAMPLE_MERGING_SUPPORTED`—and fallback code paths in `jdsample.c` and `jdcolor.c` handle uncommon subsampling ratios or cases where merged upsampling is not selected.

## Key Cross-References

### Incoming (who depends on this file)
- **No direct incoming cross-references**: `jdmerge.c` is not called from the rest of the engine. Its functions are used internally by the libjpeg decompression pipeline, specifically via `use_merged_upsample()` in `jdmaster.c`, which decides at startup whether to instantiate merged or fallback upsampling.
- **Indirect dependency chain**: `code/renderer/tr_image.c` → libjpeg API (`jpeg_read_scanlines`, etc.) → `cinfo->upsample->upsample` → this module's control functions.

### Outgoing (what this file depends on)
- **libjpeg internals only**: `jinclude.h`, `jpeglib.h` (public types), `jutils.c` (`jcopy_sample_rows` for spare-row drain in 2v case)
- **Platform/macro dependencies**: `SIZEOF`, `SHIFT_TEMPS`, `RIGHT_SHIFT`, `FIX` (fixed-point arithmetic macros); `MAXJSAMPLE`, `CENTERJSAMPLE` (constants); `GETJSAMPLE` (safe sample extraction); `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE` (output format indices).
- **No cross-module engine dependencies**: Completely isolated from qcommon, renderer, client, server subsystems.

## Design Patterns & Rationale

### Merged Upsampling Optimization
The file embodies a **single-responsibility optimization**: combine upsampling and colorspace conversion to amortize chroma multiplication costs. This is idiomatic for 1990s game engines where SIMD and GPU texturing were unavailable or nascent. The speedup comes from recognizing that in YCbCr color space, chroma terms (K1·Cr, K2·Cb, etc.) are constant across multiple Y samples sharing the same Cr/Cb pair.

### Fixed-Point Arithmetic
Uses `SCALEBITS=16` and `FIX()` macro to precompute integer LUTs rather than floating-point per-pixel. This avoids expensive floating-point ops per sample—critical on 1990s CPUs. The Cb→G table absorbs `ONE_HALF` rounding bias to eliminate an extra per-pixel addition.

### Vtable Dispatch with Compile-Time Specialization
The `upmethod` function pointer in `my_upsampler` is set at initialization based on `max_v_samp_factor` (via `jinit_merged_upsampler`), allowing the control routine to dispatch to the specialized inner loop without conditional logic. This pattern (vtable + early binding) avoids branch mispredictions in the hot path.

### Spare Row Buffering for 2v Case
When the caller supplies only a single output row buffer but the decompressor wants to emit two rows (2:1 vertical), the module caches the second row in `spare_row` and drains it on the next call. This avoids forcing the caller to manage double-buffering—a caller-side convenience pattern.

## Data Flow Through This File

```
Inbound: 
  YCbCr MCU data from prior decompression stages
  (3 input planes: Y, Cb, Cr; sampled per subsampling ratio)

Processing:
  1. One-time: build_ycc_rgb_table() → precompute 4 LUTs
     (Cr→R, Cb→B, Cr→G, Cb→G) for full sample range
  2. Per frame: start_pass_merged_upsample() → reset counters/buffers
  3. Per row group:
     - Control: merged_2v_upsample or merged_1v_upsample
       (handle spare row buffering, counter tracking)
     - Inner: h2v1_merged_upsample or h2v2_merged_upsample
       (fetch YCbCr, apply LUTs, write RGB directly to output)

Outbound:
  RGB samples written directly to caller-supplied output_buf
```

## Learning Notes

### Idiomatic to This Era
- **Fixed-point over float**: All arithmetic uses 16-bit scaling. Modern engines would use GPU texturing or SIMD; Quake III relies on precomputed tables.
- **Vtable dispatch for hot paths**: Rather than a conditional branch, function pointers are resolved once at init time. Eliminates per-call overhead.
- **Conditional module activation**: The entire merged path can be compiled out if `UPSAMPLE_MERGING_SUPPORTED` is undefined, allowing code size flexibility for embedded/ported builds.
- **Range limiting via lookup**: `range_limit` table clamps RGB values instead of if/max/min, saving branch misses in inner loop.

### Modern Contrast
- Modern decoders (libjpeg-turbo, modern browsers) use SIMD to process multiple pixels in parallel, eliminating the need for per-subsampling-ratio specialized functions.
- GPU-based decompression (via shader) or hardware-accelerated video codecs would replace this entire module.
- Contemporary code would use higher-level abstractions (e.g., Rust's type system or C++ generics) rather than macro-driven fixed-point arithmetic.

### Connection to Game Engine Concepts
This file demonstrates **explicit performance optimization for a known bottleneck** in the asset pipeline. Texture loading is not on the critical per-frame path, but precomputed LUTs and code specialization show that Quake III's designers were willing to trade code size and complexity for runtime throughput—a typical tradeoff in real-time graphics engines of the period.

## Potential Issues

**None clearly inferable from code.** The implementation is tight and correct:
- Fixed-point overflow cannot occur with ITU-R BT.601 coefficients on MAXJSAMPLE=255 and SCALEBITS=16.
- The spare row buffer correctly handles odd-height images and asymmetric caller buffer availability.
- Range limiting via `sample_range_limit` (populated by `jdmaster.c`) prevents out-of-bounds access.

The main limitation is architectural: this module's specialization for 2h1v/2h2v means rare subsampling ratios (e.g., 4:2:0, 1:1:1) silently downgrade to slower fallback code paths—by design, and documented in the file header.
