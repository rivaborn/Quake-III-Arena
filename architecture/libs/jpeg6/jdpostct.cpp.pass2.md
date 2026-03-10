# libs/jpeg6/jdpostct.cpp — Enhanced Analysis

## Architectural Role

This file is a postprocessing buffer controller within the vendored IJG libjpeg-6 library, used exclusively by the renderer's texture-loading pipeline. It sits between the upsampler/color-conversion stage and color quantization/reduction in JPEG decompression, managing buffering strategy and data flow. While part of an external library, it's critical to on-demand texture asset loading during level initialization and dynamic texture binding.

## Key Cross-References

### Incoming (who depends on this file)
- **Renderer** (`code/renderer/tr_image.c`): Calls the libjpeg decompression public API when loading JPEG textures via `ri.FS_ReadFile` + decompression pipeline
- **JPEG library internals** (`code/jpeg-6/jddecomp.c`, `jdmaster.c`): Initialize this controller via `jinit_d_post_controller()` during decompression setup
- All standard libjpeg API entry points flow through this module's state machine

### Outgoing (what this file depends on)
- **Virtual memory abstraction** (`cinfo->mem->access_virt_sarray`, `request_virt_sarray`, `alloc_sarray`): Abstracts strip/virtual-array allocation
- **Upsampler** (`cinfo->upsample->upsample`): Feeds raw upsampled data into this controller
- **Color quantizer** (`cinfo->cquantize->color_quantize`): Receives buffered data for reduction/quantization
- No dependencies on Q3A subsystems (fully encapsulated vendor code)

## Design Patterns & Rationale

**Strategy Pattern**: Three different `post_process_*` functions are plugged in via `post->pub.post_process_data` based on decompression mode (`JBUF_PASS_THRU`, `JBUF_SAVE_AND_PASS`, `JBUF_CRANK_DEST`). This avoids runtime conditionals on every data-processing call.

**Virtual Array Abstraction**: The `whole_image` virtual array allows the JPEG library to transparently spill full-image buffers to disk if memory is tight—Q3A's renderer can load large textures without allocating contiguous RAM. The `strip_height` is tuned to `max_v_samp_factor` for efficient upsampler output.

**One-pass vs. Two-pass Tradeoff**: 
- **One-pass**: Streams through with a small strip buffer, suitable for color precision reduction (no global color analysis needed).
- **Two-pass**: Stores entire image in virtual array first, then quantizes with full-image histogram (better dithering/posterization on limited color palettes).

This design reflects pre-2000s texture budgets: limited VRAM meant many levels loaded JPEG+8-bit indexed-color textures to fit 128MB VRAM.

## Data Flow Through This File

1. **Upsampler → Strip Buffer or Virtual Array**: `post_process_1pass` or `post_process_prepass` pulls upsampled MCU rows via `cinfo->upsample->upsample()` into a transient strip buffer
2. **One-pass Path** (no quantization needed): Upsampler output goes directly to quantizer, which emits final RGB/indexed data
3. **Two-pass Path** (quantization enabled): 
   - **First pass** (`post_process_prepass`): Accumulates rows into virtual array; quantizer scans data without emitting (histogram building)
   - **Second pass** (`post_process_2pass`): Reads rows from virtual array in order, quantizes with pre-computed palette, outputs final data
4. **Output**: Quantized/color-reduced texture ready for GPU binding

## Learning Notes

**Idiomatic to JPEG Library & Era**:
- Uses **JPP macro** (`#define JPP(args) (args)` on ANSI C compilers) for portable function-pointer declarations—common in 1990s C libraries before C99 standardization
- **Manual memory management** through callback vtable (`cinfo->mem`) rather than `malloc`; allows embedding in host allocator pools
- **ERREXIT macro** for error handling instead of exceptions (C had no exceptions)
- **JDIMENSION** typedef abstracts image dimensions (allows 16/32-bit width flexibility)

**Connection to Modern Game Engines**:
- Modern engines use GPU-resident texture decompression (BC/ASTC block compression) rather than CPU-side JPEG decompression
- Q3A loads JPEG at runtime; modern engines prefer pre-compressed `.dds` or `.ktx` assets
- The **buffering strategy** (strip/virtual-array) is foundational to streaming pipelines but rarely visible in modern engines due to GPU-side formats

**Not Idiomatic to Q3A Core**:
- Q3A's own code uses `Hunk_Alloc` and custom memory pools; this file uses libjpeg's own memory abstraction, showing Q3A left the JPEG library intact
- No Q3A-specific optimizations (e.g., no SIMD upsampling, no GPU streaming)

## Potential Issues

**Memory Fragmentation**: If `need_full_buffer=true` (two-pass quantization), `request_virt_sarray` may allocate a large contiguous virtual-memory block. On systems with heap fragmentation, this could fail even if total free RAM exists. The code checks `post->whole_image == NULL` and exits with `JERR_BAD_BUFFER_MODE`, but doesn't offer fallback to one-pass.

**Strip Height Coupling**: `strip_height = cinfo->max_v_samp_factor` (typically 2 or 3) may be inefficient if the upsampler can produce larger batches. No tuning parameter exposed.

**No Async/Streaming**: All buffering is synchronous and single-threaded. Multi-threaded texture loading would require per-thread JPEG decompression state.
