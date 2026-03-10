# code/jpeg-6/jdmaster.c — Enhanced Analysis

## Architectural Role

This file is the **orchestration hub for JPEG decompression** in the Quake III renderer's texture-loading pipeline. It performs one-time module selection and initialization when `jpeg_start_decompress` is called by `code/renderer/tr_image.c`, coordinating all upstream decompressor sub-modules (IDCT, Huffman decoder, upsampler, quantizer, color converter). The decompressor runs to completion offline (at texture load time), not per-frame, so this initialization cost is amortized across the lifetime of each loaded JPEG texture. No data passes through at runtime after initialization.

## Key Cross-References

### Incoming (who depends on this file)
- **code/renderer/tr_image.c** — sole in-engine consumer via `code/jpeg-6/jdapi.c` public API entry points (`jpeg_start_decompress`, which calls `jinit_master_decompress`)
- **code/jpeg-6/jdapi.c** — engine-facing decompressor API that calls `jinit_master_decompress` and `prepare_for_output_pass` / `finish_output_pass`
- **Renderer's texture cache (tr_image.h)** — texture metadata and OpenGL handle storage; jdmaster output becomes uploaded texture data

### Outgoing (what this file depends on)
- **All other code/jpeg-6/* modules** — selective `jinit_*` dispatch based on cinfo flags:
  - Quantizers: `jquant1.c` / `jquant2.c` (color reduction)
  - Color converter: `jdcolor.c` (YCbCr/CMYK→RGB)
  - Upsampler: `jdsample.c` (chroma-to-luma grid alignment)
  - Merged upsampler: `jdmerge.c` (optimized YCbCr→RGB+upsample fusion)
  - IDCT: `jddctmgr.c` (inverse cosine transform)
  - Huffman decoder: `jdhuff.c` (entropy decoding)
  - Coefficient controller: `jdcoefct.c` (buffer management)
  - Main controller: `jdmainct.c` (row-by-row output flow)
- **cinfo memory manager** — `cinfo->mem->alloc_small`, `realize_virt_arrays`
- **Math utilities** — `jdiv_round_up` (from jutils.c)

## Design Patterns & Rationale

### 1. **Conditional Module Composition**
The `use_merged_upsample()` predicate and subsequent `if (master->using_merged_upsample)` branching demonstrate **runtime capability negotiation**. Instead of always assembling separate upsample/color-convert modules, this checks:
- Sampling factors (2h1v or 2h2v only)
- Color space (YCbCr→RGB only)
- Upsampling style (no fancy filtering)
- IDCT scaling uniformity

**Why:** The merged path (`jdmerge.c`) is faster for common cases but less flexible. Conditional selection defers to caller's constraints.

### 2. **Dummy-Pass Mechanism for Multi-Mode Quantization**
The `is_dummy_pass` flag allows 2-pass quantization to run the full pipeline twice without rewriting it:
- First pass (dummy): accumulate histogram of colors
- Second pass (real): use final palette to reduce colors

**Why:** Avoids code duplication. A single pipeline can serve multiple quantization strategies by toggling buffer modes (`JBUF_CRANK_DEST`, `JBUF_SAVE_AND_PASS`, `JBUF_PASS_THRU`) between passes.

### 3. **Deferred Output Dimension Calculation**
`jpeg_calc_output_dimensions()` is exported and safe to call multiple times (idempotent, no side effects besides output field writes). This is called both by `master_selection()` and potentially by the application before `jpeg_start_decompress()`.

**Why:** Lets the caller query output size (e.g., for buffer pre-allocation) before committing to decompression. Decouples dimension discovery from module initialization.

### 4. **Pre-IDCT Range Limiting via Lookup Table**
The `sample_range_limit` table uses a clever offset-and-wraparound strategy: corrupt post-IDCT values are masked (`x & MASK`) and safely index into the table, producing bogus-but-in-range output instead of buffer overrun.

**Why:** This is a 1990s-era micro-optimization. On CPUs without branch prediction, table lookup (`x = table[x]`) was faster than conditional tests (`if (x < 0) x = 0`). The masking technique handles malformed JPEG input defensively.

## Data Flow Through This File

1. **Entry:** `jinit_master_decompress(cinfo)` is called once by `jpeg_start_decompress` (from `jdapi.c`).
2. **Dimension Setup:** `jpeg_calc_output_dimensions()` computes output width/height and per-component DCT scaling.
3. **Infrastructure:** `prepare_range_limit_table()` allocates and populates the sample clipping lookup.
4. **Module Cascading:** `master_selection()` conditionally calls `jinit_*` on ~10 sub-modules in dependency order:
   - Entropy decoder (Huffman or arithmetic)
   - Inverse DCT
   - Upsampler(s) and color converter
   - Quantizer (0, 1, or 2 instances depending on mode)
   - Output controllers (post, main, coef)
5. **Per-Pass Lifecycle:** Once decompression begins, `prepare_for_output_pass()` and `finish_output_pass()` bracket each output pass, dispatching `start_pass` and `finish_pass` to active sub-modules.
6. **Output:** Pixel data flows out via `cinfo->main` controller back to the renderer's texture cache.

## Learning Notes

### Idiomatic to JPEG Library & 1990s Engine Practice
- **Compile-time feature flags** (`UPSAMPLE_MERGING_SUPPORTED`, `IDCT_SCALING_SUPPORTED`, `D_PROGRESSIVE_SUPPORTED`, `QUANT_1PASS_SUPPORTED`): Allow vendors to configure the library at build time. Modern engines use runtime feature detection or always enable all features.
- **Virtual array realization** (`realize_virt_arrays`): The JPEG library pre-allocates virtual (disk-backed) arrays to handle large progressive images. Modern decoders stream pixel-by-pixel.
- **Buffered-image mode**: Supports pausing decompression mid-stream and resuming later, or re-using results with different output settings (e.g., different quantization on subsequent passes). Most modern APIs decompress in one shot.
- **Range-limit table optimization**: A classic example of "avoid branches with data-dependent lookup tables" — a principle that has largely been superseded by modern CPU branch prediction and SIMD.

### Modern Engine Pattern Contrast
Modern game engines typically:
- Use hardware-accelerated image decoders (DirectX, Metal, Vulkan native JPEG decoders) or simple streaming libraries
- Avoid compile-time feature flags; always include all capabilities
- Decompress asynchronously on worker threads
- Cache decompressed pixels in GPU memory immediately

Quake III's approach is typical of the id Tech 3 era: lightweight, self-contained, deterministic, and no dynamic library dependencies.

## Potential Issues

- **Arithmetic coding not supported:** `JERR_ARITH_NOTIMPL` is always thrown if a JPEG uses arithmetic coding. This was disabled to reduce code size and avoid patent complications (arithmetic coding was patent-encumbered at the time). Modern JPEG-XL and AVIF have superseded this, but old JPEGs with arithmetic coding will fail to load.
- **Width overflow check:** Line ~355 checks `(long) jd_samplesperrow != samplesperrow` to catch scanline row size exceeding `JDIMENSION` max. This guards against integer overflow but is a rare corner case for 2005-era textures.
- **Quantizer initialization cost:** If 2-pass quantization is enabled, the first pass incurs overhead (histogram accumulation). For single-pass quantization or external colormaps, this is skipped. The selection logic (lines ~320–347) determines which path is taken based on `enable_*quant` flags.

---

## Summary

**jdmaster.c is a static, offline orchestrator.** It runs once at JPEG decompression startup (during texture load), assembles a pipeline of sub-modules, and then steps aside. The file demonstrates **conditional architecture composition** (merged upsampler, multi-pass quantization) and **1990s micro-optimizations** (range-limit lookup table, compile-time features). From the engine's perspective, this is transparent infrastructure: the renderer calls the JPEG API, and pixels emerge in the texture cache. No per-frame cost, no dynamic data flow beyond initialization.
