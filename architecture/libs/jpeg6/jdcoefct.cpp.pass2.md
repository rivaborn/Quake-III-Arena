# libs/jpeg6/jdcoefct.cpp — Enhanced Analysis

## Architectural Role

This file is a vendored component of IJG libjpeg-6, providing the coefficient buffer controller that bridges entropy decoding (Huffman stream parsing) and inverse-DCT spatial reconstruction. It is used exclusively by the renderer's texture loader (`code/renderer/tr_image.c` → `jload.c`) for runtime decompression of JPEG textures referenced in BSP files. The controller abstracts single-pass vs. multi-pass operation, allowing both memory-efficient streaming decompression and full-image buffering for progressive JPEG decoding with optional block smoothing.

## Key Cross-References

### Incoming (who depends on this file)
- **JPEG public API (`jinclude.h`, `jpeglib.h`)**: Public functions exposed via `jpeg_decompress_struct` vtable for texture loading pipeline
- **Renderer texture loader** (`code/renderer/tr_image.c` via vendored `jload.c`): Only runtime consumer; loads BSP texture references as JPEG, decompresses via this coefficient controller

### Outgoing (what this file depends on)
- **Entropy decoder** (`cinfo->entropy->decode_mcu`): Parses Huffman-encoded MCU blocks
- **IDCT functions** (`cinfo->idct->inverse_DCT[]`): Component-specific inverse-DCT transforms (one method per component index)
- **JPEG memory manager** (`cinfo->mem->access_virt_barray`, `alloc_small`): Virtual array allocation for multi-pass, latch buffer for smoothing
- **Input control** (`cinfo->inputctl->consume_input`, `finish_input_pass`, `eoi_reached`): Synchronization for multi-pass operation

## Design Patterns & Rationale

**Pluggable coefficient controller via vtable**: The `my_coef_controller` struct extending `jpeg_d_coef_controller` allows the decompressor core to swap single-pass vs. multi-pass implementations. This reflects the library's goal of supporting both memory-constrained (embedded) and feature-rich (workstation) decompression modes.

**State machine for MCU enumeration**: `MCU_ctr`, `MCU_vert_offset`, and `MCU_rows_per_iMCU_row` track position within an iMCU row for suspension safety—if the entropy decoder blocks (no more input), the controller can resume at the exact MCU without re-parsing.

**Virtual arrays for progressive JPEG**: Multi-pass modes (`#ifdef D_MULTISCAN_FILES_SUPPORTED`) buffer the full coefficient image using the JPEG memory manager's swappable virtual array layer, allowing images larger than RAM. Single-pass (`decompress_onepass`) minimizes footprint with a one-MCU workspace.

**Block smoothing as optional post-processing**: The smoothing logic estimates missing AC coefficients from neighboring DC values during progressive JPEG decoding, reducing blockiness artifacts. It is conditional on `cinfo->progressive_mode` and `cinfo->do_block_smoothing`, decoupling it from core decompression.

**Lazy reachability checks**: The controller defers full knowledge of component sampling factors and block dimensions until `start_iMCU_row`, allowing late binding of interleaved vs. non-interleaved scan configuration.

## Data Flow Through This File

1. **Input**: Entropy decoder fills `MCU_buffer[D_MAX_BLOCKS_IN_MCU]` with decoded DCT coefficients (one MCU per call)
2. **Processing**:
   - Single-pass: immediately invoke component-specific inverse-DCT on each block, write samples to output plane
   - Multi-pass: store coefficients in virtual `whole_image[]` arrays, deferred inverse-DCT in output pass
3. **Output**: Spatial-domain samples written to `output_buf[component][row][col]`, consumed by renderer for texture binding

**Synchronization boundary**: In multi-pass mode, `decompress_data` forces input to stay at least one iMCU row ahead (for DC scan lookahead in smoothing). The `input_scan_number`, `output_scan_number` counters synchronize separate input and output passes.

## Learning Notes

**Idiomatic to this era (1994–1995)**:
- Explicit state counters instead of iterators; no callback-based streaming
- `FAR` keyword for segmented memory (80x86 real-mode holdover, harmless on modern platforms)
- Macro-based configuration (`#ifdef D_MULTISCAN_FILES_SUPPORTED`, `#ifdef BLOCK_SMOOTHING_SUPPORTED`) rather than runtime feature flags
- Hand-rolled suspension via return codes (`JPEG_SUSPENDED`, `JPEG_ROW_COMPLETED`, `JPEG_SCAN_COMPLETED`) predating generator/coroutine abstractions

**Modern engines differ**:
- Stream-based decoders (e.g., WebP, HEIF) use progressive callbacks or pull-based iterators
- GPU-accelerated JPEG decode (NVDEC, QuickSync) bypasses software decompression entirely
- Progressive rendering can interleave coefficient buffering with inverse-DCT, not strict phase separation

**Connection to engine architecture**: This file exemplifies how Quake III isolates high-complexity vendor code (libjpeg) behind a minimal public interface. The renderer never directly references internal JPEG types—all interaction flows through `jload.c`, which translates the public API into `image_t` (renderer's internal texture representation). This separation allows easy swapping of JPEG implementations without touching engine code.

## Potential Issues

**None clearly identifiable** from the provided context. The code follows IJG's well-tested patterns. However:
- **Buffer overrun risk in block smoothing**: The `coef_bits_latch` allocation assumes `cinfo->num_components * (SAVED_COEFS * sizeof(int))`. If `num_components` exceeds design limits (typically 4 for CMYK), overallocation could occur—though the JPEG spec caps components at 255, practical images rarely exceed 4.
- **Suspension safety**: Suspension during inverse-DCT (not between MCUs) could corrupt `output_buf`. The code guards this by only invoking IDCT within `yindex < useful_width` bounds checks, but relies on correct caller initialization of `output_buf`.
- **Virtual array swapping**: Multi-pass mode assumes the memory manager can spill to disk. If virtual arrays exhaust storage, `access_virt_barray` may silently fail or return NULL, causing a dereference crash. Caller (`decompress_data`) does not validate `buffer != NULL`.
