# code/jpeg-6/jdpostct.c — Enhanced Analysis

## Architectural Role

This file implements a critical buffering stage within the vendored **libjpeg-6 decompression pipeline**. It bridges upsampling/color-conversion and color quantization—two fundamentally different memory-access patterns (streaming vs. histogram). The engine loads all textures via JPEG, making this module's efficiency directly impact texture streaming performance during level load and runtime GPU uploads. The module embodies a classic **two-pass algorithm trade-off**: single-pass quantization (low latency, predictable memory) vs. two-pass (high quality, requires full-image buffering).

## Key Cross-References

### Incoming (who depends on this file)
- **Other JPEG library files** (`jdapistd.c`, `jdmaster.c`, etc.) call `jinit_d_post_controller` during decompression initialization and invoke the selected `post_process_data` function pointer during output.
- **Renderer (`code/renderer/tr_image.c`)** indirectly depends on this via `jload.c`, which drives the full JPEG decompression pipeline to load texture pixels from `.jpg` asset files.

### Outgoing (what this file depends on)
- **JPEG memory allocator** (`cinfo->mem->*` function pointers): `alloc_small`, `alloc_sarray`, `request_virt_sarray`, `access_virt_sarray`. These are bound to the engine's custom allocators at JPEG init time, allowing the library to work within the engine's memory pools.
- **Upsampler module** (`cinfo->upsample->upsample`): delegates sampling logic; this module only orchestrates buffering around it.
- **Color quantizer module** (`cinfo->cquantize->color_quantize`): receives buffered pixel data; one-pass vs. two-pass paths differ in whether output is emitted.
- **JPEG state machine** (`cinfo` fields like `quantize_colors`, `output_height`, `max_v_samp_factor`): reads configuration that determines which code path to take.

## Design Patterns & Rationale

### Two-Pass Quantization with Virtual Arrays
The module demonstrates a **deferred I/O pattern**: when two-pass color quantization is enabled (`need_full_buffer=TRUE`), the entire decompressed image is buffered in a virtual array (`whole_image`), allowing a histogram pass to build a custom color palette before the second quantization pass. This trades **memory bandwidth and latency** (buffering full image to disk/VM) for **output quality** (globally optimal palette). The engine must decide at initialization time which mode to use based on `need_full_buffer` (typically `FALSE` for real-time texture loading).

### Strip Buffer as Pipeline Breaker
For single-pass mode (`need_full_buffer=FALSE`), a **strip buffer** (`post->buffer`) sized to `max_v_samp_factor` rows decouples upsampling granularity from quantization/output granularity. This allows the upsampler to return rows at its natural stride without forcing the quantizer to emit output at the same rate—a classic **producer/consumer buffer** pattern.

### Conditional Bypass via Function Pointers
When `quantize_colors=FALSE`, the entire module is **short-circuited** in `start_pass_dpost`: `post_process_data` is set directly to `cinfo->upsample->upsample`, eliminating the postprocessor's overhead. This avoids a redundant buffering layer for quality (non-reduced) paths. The indirection through function pointers allows dynamic routing without conditional branches in the hot loop.

## Data Flow Through This File

1. **Initialization** (`jinit_d_post_controller`):
   - Allocates controller struct from `JPOOL_IMAGE`.
   - If quantization needed: allocates either a full-image virtual array (two-pass) or a strip buffer (one-pass).
   - Register's `start_pass_dpost` as the pass-initialization callback.

2. **Pass Setup** (`start_pass_dpost`):
   - Inspects pass mode (`JBUF_PASS_THRU`, `JBUF_SAVE_AND_PASS`, `JBUF_CRANK_DEST`).
   - Selects function pointer: `post_process_1pass`, `post_process_prepass`, `post_process_2pass`, or bypass to upsampler.
   - Resets strip row counters.

3. **One-Pass Path** (`post_process_1pass`):
   - Upsample into strip buffer up to `min(output_available, strip_height)`.
   - Immediately quantize and emit to output scanline buffer.
   - No row tracking beyond the current strip.

4. **Two-Pass Prepass** (`post_process_prepass`):
   - Upsample into virtual array, advancing through full image by strip boundaries.
   - Call quantizer in **histogram-only mode** (output=NULL); quantizer accumulates color statistics.
   - Emit nothing to output; only advance `out_row_ctr` so outer loop doesn't stall.

5. **Two-Pass Output Pass** (`post_process_2pass`):
   - **No upsampling** in this pass (input buffers unused).
   - Re-read quantized rows from virtual array (read-only access).
   - Quantizer outputs final indexed pixels with the palette built in prepass.
   - Must manually check image bottom since upsampler unavailable.

## Learning Notes

**Idiomatic Two-Pass Quantization**: This module teaches the classical approach to high-quality color quantization in memory-constrained environments (pre-2000s). Modern engines typically use:
- GPU texture compression (BC1–BC7) instead of runtime quantization.
- Single-pass dithering or pre-computed palettes.
- Hardware-native color spaces (sRGB, 16-bit half-float).

However, the **buffering abstraction** (strip vs. full-image virtual arrays) and **pass mode dispatch** remain relevant for streaming decompression pipelines today.

**Virtual Array Abstraction**: The JPEG library's `jvirt_sarray_ptr` encapsulates paging to disk or memory. The engine's allocator can intercept this to use its hunk/zone system. This is an example of **dependency injection** allowing a third-party library to respect the engine's memory topology—critical for embedded/console systems with rigid memory layouts.

**Pipeline Stage Ownership**: Unlike modern engines where decompression often runs on worker threads, this synchronous buffering stage makes assumptions about single-threaded execution and sequential row access. Texture streaming in modern Q3 forks decompression to background tasks; the buffer ownership here would require thread-safe virtual arrays.

## Potential Issues

1. **Virtual Array Overhead for Two-Pass**: Buffering full decompressed image (`output_width × output_height × out_color_components` bytes) for a large (e.g., 4K) texture can exceed available hunk memory, forcing disk paging. No fallback to one-pass degradation if two-pass allocation fails.

2. **Output Height Not Checked in One-Pass**: `post_process_1pass` relies on the upsampler to detect image bottom (`old_next_row >= cinfo->output_height`); if upsampler is buggy, row overflow is possible. Two-pass explicitly checks `cinfo->output_height` to prevent this.

3. **Memory Pool Binding**: All allocations come from `JPOOL_IMAGE`, shared across all JPEG instances. Parallel texture loading could cause contention; no per-instance pool isolation visible.

4. **No Progress Callback in Two-Pass Prepass**: First pass emits no output pixels, only advances counters. Long prepass on massive images could make the engine appear frozen to the user if no UI feedback is provided.
