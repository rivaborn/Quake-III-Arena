# code/jpeg-6/jdcoefct.c — Enhanced Analysis

## Architectural Role
This file implements the coefficient buffering and IDCT coordination layer within the vendored IJG libjpeg-6 library used exclusively by the Renderer subsystem (`code/renderer/tr_image.c`) for texture decompression. It bridges the decoder pipeline's entropy stage and output transform stage, abstracting away the buffering strategy (single-pass streaming vs. multi-pass virtual arrays) from the IDCT machinery. The file's conditional compilation gates (`D_MULTISCAN_FILES_SUPPORTED`, `BLOCK_SMOOTHING_SUPPORTED`) reflect design choices made at library compile-time, baking in the specific JPEG features Q3A's renderer requires.

## Key Cross-References

### Incoming (Renderer Dependency)
- Called indirectly through libjpeg's decompression API: `jpeg_read_scanlines()` (or equivalent), which invokes functions wired up by `jinit_d_coef_controller`
- The Renderer (`code/renderer`) never calls this file directly; it uses the public libjpeg interface (`jpeglib.h`), which hides internal buffering
- No direct interdependencies with other Q3A engine subsystems; entirely isolated within the vendored library

### Outgoing (Dependencies Within libjpeg)
- **Entropy decoder**: calls `(*cinfo->entropy->decode_mcu)()` to pull MCU coefficient blocks
- **Memory manager**: calls `cinfo->mem->access_virt_barray()` for virtual image array access and `cinfo->mem->alloc_small/alloc_large()` for structure allocation
- **IDCT implementations**: dispatches to `cinfo->idct->inverse_DCT[component]` function pointers (specific transforms defined elsewhere in `code/jpeg-6`)
- **Input controller**: calls `cinfo->inputctl->consume_input()` and `finish_input_pass()` for scan boundary coordination
- **Quantization tables**: reads from `cinfo->comp_info[ci]->quant_table` for smoothing validation

## Design Patterns & Rationale

**Conditional Compilation (Feature Gates)**:
- `D_MULTISCAN_FILES_SUPPORTED`: gates full-image virtual arrays for progressive/multi-scan mode
- `BLOCK_SMOOTHING_SUPPORTED`: gates K.8 interblock smoothing for progressive decompression
- This compile-time stratification is typical of IJG to reduce footprint: single-pass-only builds omit multi-pass code entirely

**Function Pointer Dispatch** (`decompress_data` field):
- Allows `start_output_pass()` to swap implementations (e.g., `decompress_smooth_data` vs. `decompress_data`) at runtime without rebuilding
- Mirrors Quake III's own vtable pattern used throughout (e.g., Renderer's `refexport_t`, QVM dispatcher)
- Avoids branching overhead in the hot path; once chosen, the vtable persists for the pass

**Two-Path Architecture**:
- **Single-pass** (`decompress_onepass`): input and output run in lockstep; requires only a one-MCU buffer; used for streaming/baseline JPEG
- **Multi-pass** (`consume_data` / `decompress_data`): decodes entire scan into virtual arrays before output, enabling progressive refinement and alternative routing (K.8 smoothing)
- The choice is made at initialization (`need_full_buffer` param to `jinit_d_coef_controller`) based on whether the decompressor is in buffered-image mode

**Virtual Buffer Abstraction**:
- `cinfo->mem->access_virt_barray()` manages on-demand paging of DCT coefficient blocks to disk/memory as needed
- Decouples MCU buffering from physical memory constraints, critical for large images
- Likely essential for platforms with limited RAM when decoding high-resolution textures

## Data Flow Through This File

1. **Input Side (Entropy Decoding)**:
   - `start_input_pass()` resets per-scan counters
   - Per iMCU row: `consume_data()` calls entropy decoder and writes MCU blocks into `whole_image` virtual arrays
   - Blocks organized by component and spatial position for later IDCT traversal

2. **Output Side (IDCT & Emission)**:
   - `start_output_pass()` may select smoothing variant if progressive + applicable
   - Per iMCU row: `decompress_data()` reads blocks from virtual arrays, applies IDCT per block, writes samples to output planes
   - Synchronization: output waits for input to reach safe positions via `consume_input()` call

3. **Interblock Smoothing (Progressive Only)**:
   - `smoothing_ok()` validates that quantization tables exist and DC values are known
   - `decompress_smooth_data()` uses a 3×3 sliding window of DC values to estimate missing AC01, AC10, AC20, AC11, AC02 coefficients
   - Follows JPEG Annex K.8; reduces blocking artifacts in progressive decompression
   - Modified coefficients stay in a local workspace; original virtual array untouched

## Learning Notes

**Idiomatic to This Library Era (1994–1995)**:
- Heavy use of `FAR` qualifiers (`void FAR *`) for 16-bit segmented-memory x86; Q3A still honors this in the vendored code
- Macro-heavy: `JPP()` for function declarations, `METHODDEF`/`LOCAL` for visibility control
- No heap abstractions or leak detection; all allocation flows through `cinfo->mem` vtable for centralized tracking
- Verbose naming: `MCU_vert_offset`, `MCU_rows_per_iMCU_row` — precision over brevity, reflecting 1990s C style

**Modern Engine Differences**:
- Most modern engines use hardware texture decompression (GPU) or precompiled texture formats (DDS, PKM, ASTC)
- Quake III still decodes JPEG on CPU, then uploads to GPU; this buffering layer reflects that choice
- Progressive JPEG support (K.8 smoothing) is rare in modern games; most use baseline or fully lossless formats
- Virtual memory paging is obsolete; modern builds would likely just allocate heap directly

**Engine Integration Context**:
- This file is *entirely isolated* within the vendored library—no Quake III engine code imports its symbols
- The Renderer's texture loading (`tr_image.c`) talks only to the public libjpeg API (`jpeg_read_scanlines`, etc.)
- Compile-time configuration (`D_MULTISCAN_FILES_SUPPORTED`, etc.) is chosen once at build time, baking in feature support
- No runtime plugin or swappable image codec system; JPEG support is hard-coded and permanent

## Potential Issues

**None clearly inferable from provided context**, but structurally:
- **Progressive mode + no buffering**: if `D_MULTISCAN_FILES_SUPPORTED` is undefinialized but a progressive JPEG is encountered, `jinit_d_coef_controller` will error out (`ERREXIT`) rather than degrade gracefully
- **FAR pointers on 32-bit+**: legacy `FAR` qualifiers are effectively no-ops and may hide portability issues on systems where they once mattered
- **No bounds checking in smoothing**: `decompress_smooth_data` assumes the 3×3 DC window exists; edge blocks rely on careful comparison logic to avoid buffer overruns
