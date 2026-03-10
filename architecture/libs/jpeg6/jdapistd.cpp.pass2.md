# libs/jpeg6/jdapistd.cpp — Enhanced Analysis

## Architectural Role

This file implements the **standard decompression API** for the vendored IJG libjpeg-6 library, which the engine uses exclusively for loading JPEG textures at runtime. It provides the public-facing entry points (`jpeg_start_decompress`, `jpeg_read_scanlines`, `jpeg_read_raw_data`) consumed by the **renderer's texture loader** (`code/renderer/tr_image.c` → `code/jpeg-6/jload.c`). The file bridges low-level decompression mechanics (multi-scan input absorption, quantization passes) with the scanline-oriented output expected by image upsampling and GPU upload.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/jpeg-6/jload.c`** – The sole runtime caller; texture loading wrapper that invokes `jpeg_start_decompress()` → `jpeg_read_scanlines()` → decompressor cleanup
- **`code/renderer/tr_image.c`** – Indirectly, via `jload.c`; initiates JPEG texture loads during asset precaching
- Offline tool **`q3map` / `q3radiant`** may also use libjpeg for texture processing, but this is not instrumented in the runtime engine cross-reference

### Outgoing (what this file depends on)
- **`code/jpeg-6/jinclude.h`, `jpeglib.h`** – IJG public API and internal type definitions; relies entirely on the cohesive libjpeg-6 data structures (`j_decompress_ptr`, `cinfo->master`, `cinfo->main`, etc.)
- **Implicitly on other libjpeg modules** – Functions like `jinit_master_decompress()`, `(*cinfo->inputctl->consume_input)()`, `(*cinfo->main->process_data)()` are dispatched through function pointers set up elsewhere in the library (e.g., `jdapimin.c` during `jpeg_read_header()`)
- **No outgoing calls to engine** – This file is pure decompression logic; zero calls back to qcommon, renderer, or any Q3 subsystem

## Design Patterns & Rationale

**State Machine via `global_state`:**  
The decompressor tracks a strict lifecycle (`DSTATE_READY` → `DSTATE_PRELOAD` → `DSTATE_PRESCAN` → `DSTATE_SCANNING`/`DSTATE_RAW_OK`). This enforces call sequencing and detects application programmer errors (e.g., calling `jpeg_read_scanlines()` before `jpeg_start_decompress()`).

**Conditional Multi-Scan Absorption:**  
When `has_multiple_scans` is true and `D_MULTISCAN_FILES_SUPPORTED` is defined, the decompressor loops through all input scans upfront (lines 53–75), absorbing them into the coefficient buffer. This enables two-pass quantization on hardware without sufficient memory. The progress monitor hook is called periodically to allow UI responsiveness during long loads.

**Dummy Pass Pattern (lines 107–128):**  
Two-pass quantization (e.g., octree or median-cut) requires a first pass to build the color palette and a second to emit indexed pixels. The `output_pass_setup()` function orchestrates dummy passes transparently: if `is_dummy_pass` is set, it drives `process_data()` with a NULL output buffer to accumulate statistics, then resets for the real pass.

**Rationale:** Separates concerns — the decompression state machine doesn't care *why* multiple passes exist; it just respects the master control's directives. This allows the library to handle both single-pass (direct RGB output) and multi-pass (quantized palette) scenarios without branching in the API.

**Graceful Suspension:**  
Return values of `FALSE` (lines 69, 77, 169, 195) indicate the data source is exhausted without error, allowing incremental streaming or pause/resume. This is idiomatic for library code but rarely exercised by the renderer, which typically supplies complete JPEG data upfront.

## Data Flow Through This File

```
jpeg_read_header() [jdapimin.c]
         ↓
jpeg_start_decompress()
  • Initialize master control (jmaster)
  • Handle buffered_image mode branching
  • If has_multiple_scans: drain all input scans
         ↓
output_pass_setup()
  • Call prepare_for_output_pass
  • Loop: while is_dummy_pass, drive process_data with NULL
         ↓
jpeg_read_scanlines() [texture loader loop]
  • Call process_data() with output buffer
  • Increment output_scanline
  • Return rows consumed
         ↓
jpeg_finish_decompressor() [cleanup, in jdapimin.c]
```

**Key state variables:**
- `cinfo->global_state` – Enforces call sequencing
- `cinfo->output_scanline` – Current Y position in output image
- `cinfo->progress` – Optional progress callback (used by tools, not renderer)

## Learning Notes

**Idiomatic IJG Library Design:**
This code exemplifies the IJG libjpeg-6 philosophy of modular, callback-driven architecture. The decompressor doesn't allocate output buffers; the caller provides them. The decompressor doesn't know the output format; it dispatches to `cinfo->main->process_data()`, which is set by the master control based on compile-time and runtime options (e.g., color space, subsampling).

**Modern Engines Would:**
- Use a stateless streaming API (e.g., libvpx's `vpx_codec_decode()`) or lazy evaluation (WebP's incremental decoding)
- Separate I/O (pull-based data source) from processing
- Return error codes instead of `longjmp` on fatal errors
- Provide async/SIMD variants for performance-critical paths

**Why This Persists in Q3A:**
Libjpeg-6 was the de facto standard in 1999–2005 and remains stable. The engine vendored it to avoid link-time dependencies and ensure consistent behavior across platforms. Its multi-pass quantization was valuable for software rendering on constrained hardware (early 2000s GPUs had limited color precision).

## Potential Issues

1. **Memory starvation on very large JPEGs:**  
   Line 53–75 absorbs *all* input scans into `cinfo->coef` buffer (coefficient storage) if `has_multiple_scans` is set. A pathological JPEG with hundreds of scans could exhaust the hunk buffer. Unlikely in practice (modern JPEGs use baseline/progressive, not multi-scan), but not defended.

2. **Missing `jpeg_finish_decompressor()`:**  
   This file doesn't call it; cleanup happens in the caller (`jload.c`). If `jload.c` omits cleanup, the coefficient buffer is never released. (Low risk; the renderer consistently cleans up.)

3. **Progress monitor overhead:**  
   Lines 61–62 and 115–117 call the progress hook on every scan/scanline. For textures loaded in real-time, this per-frame overhead is negligible, but for batch texture conversion tools, it could be measurable. Not a correctness issue, just performance.
