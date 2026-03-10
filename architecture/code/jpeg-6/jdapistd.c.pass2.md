# code/jpeg-6/jdapistd.c — Enhanced Analysis

## Architectural Role

This file implements the output-phase API for Quake III's vendored JPEG decompressor, sitting at a critical junction in the texture loading pipeline. When the renderer's image loader (`tr_image.c`) encounters a `.jpg` texture, it invokes these functions to transform the compressed bitstream (parsed by `jdapimin.c`) into raw pixel data for upload to OpenGL. The file's separation from `jdapimin.c` is deliberate: it allows transcoder-only builds (which recompress JPEGs without display) to avoid linking the full decompressor—a constraint that mattered in the late 1990s when binary size was a scarce resource. By managing the decompressor's state machine and driving the output pipeline, this file bridges the external API (what the renderer sees) and internal infrastructure (master controller, coefficient decoder, quantizer).

## Key Cross-References

### Incoming (who depends on this file)
- **`code/renderer/tr_image.c`** — The primary consumer; calls `jpeg_read_scanlines` (or `jpeg_read_raw_data` for raw decoding) during `R_LoadJPG` to decompress JPEG texture assets
- Any engine code that loads textures with `.jpg` extension indirectly depends on these functions
- The texture caching and mipmap generation pipeline in the renderer depends on this API to convert compressed JPEGs to raw RGB that can be resampled and uploaded

### Outgoing (what this file depends on)
- **`jdmaster.c`** (same directory) — Provides `jinit_master_decompress` and the master control vtable (`prepare_for_output_pass`, `finish_output_pass`, `is_dummy_pass`)
- **`jdinput.c`** — Input controller; provides the `consume_input` function pointer and manages the compressed bitstream
- **`jdmainct.c`** — Main processor; provides `process_data` function pointer that converts coefficient data to scanlines
- **`jdcoefct.c`** — Coefficient decoder; provides `decompress_data` for raw iMCU-row output
- **`jpeglib.h` / `jinclude.h`** — Foundational types and macros from the JPEG library's public interface

## Design Patterns & Rationale

- **Deterministic State Machine**: The `DSTATE_*` enumeration (`DSTATE_READY`, `DSTATE_PRELOAD`, `DSTATE_PRESCAN`, `DSTATE_SCANNING`, `DSTATE_RAW_OK`, `DSTATE_BUFIMAGE`, `DSTATE_BUFPOST`) explicitly models the decompressor's lifecycle. This is critical for **suspension/resumption**: a function can return FALSE when input stalls, and the caller simply re-invokes it later without re-initialization. The global state is implicitly preserved in `cinfo`.

- **Dual-Output Modes**: Scanline-at-a-time (`jpeg_read_scanlines`) and raw iMCU-row (`jpeg_read_raw_data`) decoding are decoupled. This allows the renderer to choose between efficient scanline iteration and raw component data (for specialized effects or precise control).

- **Buffered-Image Mode**: Progressive JPEGs can be decoded incrementally via `jpeg_start_output` / `jpeg_finish_output` per scan, enabling incremental rendering. This is guarded by `D_MULTISCAN_FILES_SUPPORTED`, allowing the compiler to omit the feature in space-constrained builds.

- **Dummy Passes for Quantization**: When color quantization is enabled (`QUANT_2PASS_SUPPORTED`), `output_pass_setup` runs a first pass with NULL output buffer (to build a color histogram) before the final pass produces pixels. This is a space-efficient way to support posterization/palette reduction without buffering the entire image.

- **Deliberate File Separation** (`jdapimin.c` vs. `jdapistd.c`): The comment at the top explains the rationale: `jdapimin.c` handles minimal header parsing; if transcoder code links only that, it avoids the full decompressor. This is an artifact of the late-1990s concern with binary bloat.

## Data Flow Through This File

1. **Initialization** (`jpeg_start_decompress`):
   - Input: `cinfo` in `DSTATE_READY` after `jpeg_read_header` has populated metadata
   - Master decompressor is initialized via `jinit_master_decompress`
   - If multi-scan (progressive): all scans are absorbed into `cinfo->coef` via `consume_input` loop (buffered-image mode exits early)

2. **Dummy Passes** (if quantization enabled):
   - `output_pass_setup` calls `prepare_for_output_pass` and iteratively invokes `process_data` with NULL output buffer
   - `output_scanline` advances without producing pixels; used to histogram colors for palette reduction

3. **Output Modes**:
   - **Scanline**: Caller loops `jpeg_read_scanlines`, reading up to `max_lines` decompressed rows per call; `output_scanline` advances automatically
   - **Raw**: Caller invokes `jpeg_read_raw_data` to extract one iMCU row of unsampled component data directly

4. **Buffered-Image Flow** (progressive JPEGs):
   - Outer loop: `jpeg_start_output(scan_number)` → read scanlines/raw → `jpeg_finish_output` → repeat for next scan
   - Each call to `start_output` positions the decoder at a specific scan; `finish_output` advances the input controller past markers

5. **Exit**: Functions return FALSE on suspension (input exhausted) or TRUE on completion

## Learning Notes

- **Vendored Third-Party Code**: This is IJG libjpeg-6 (circa 1994–1995), one of the most widely-ported JPEG decoders ever written. Quake III vendors it to avoid external dependencies; modern engines might use libjpeg-turbo (faster), libpng, or platform-native decoders.

- **Suspension without Callbacks**: The resumable-function pattern (returning FALSE, implicit state in `cinfo`) is elegant for resource-constrained systems. It predates callback-based or async-await models and requires the caller to implement retry logic—a trade-off between complexity and flexibility.

- **Portable C from the 1990s**: The proliferation of typedefs (`JDIMENSION`, `JSAMPARRAY`, `JSAMPIMAGE`) reflects the library's portability to 16-bit and 32-bit machines with different calling conventions. Modern C assumes 32/64-bit machines and would simplify these.

- **Progress Monitoring**: The `progress_monitor` callback pattern (checking `!= NULL` before calling) was a common idiom before exceptions or error-code returns. The renderer could use this to update a "Loading..." UI, though it likely doesn't in Quake III.

- **Conditional Compilation Overhead**: The `#ifdef D_MULTISCAN_FILES_SUPPORTED` and `#ifdef QUANT_2PASS_SUPPORTED` blocks are a form of feature gating. They reduce code size and complexity for builds that don't need those features—at the cost of compile-time brittleness (mismatched flags between library and app cause silent failures).

## Potential Issues

- **Conditional Compilation Fragility**: If `D_MULTISCAN_FILES_SUPPORTED` or `QUANT_2PASS_SUPPORTED` are defined differently in this file than in the app invoking it, progressive JPEGs or quantized images may silently fail to decompress (hitting `ERREXIT` with "not compiled").

- **Buffer Size Assumptions in Raw Mode**: `jpeg_read_raw_data` validates `max_lines >= lines_per_iMCU_row` and ERREXITs if too small. However, it assumes the caller's buffer is actually that large. Miscalculation by the caller risks buffer overrun.

- **No Thread Safety**: The file assumes single-threaded access to `cinfo`. If multiple threads call these functions on the same context, data corruption is likely.

- **State Retention Across Calls**: The suspension/resumption mechanism relies on `cinfo->global_state` persisting. If the caller inadvertently overwrites or resets `cinfo`, or uses it from multiple threads, the state machine will malfunction.
