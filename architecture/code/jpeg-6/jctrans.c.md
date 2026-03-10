# code/jpeg-6/jctrans.c

## File Purpose
Implements JPEG transcoding compression: writing pre-existing raw DCT coefficient arrays directly to an output JPEG file, bypassing the normal pixel-data compression pipeline. Also provides utilities for copying critical image parameters from a decompression source to a compression destination.

## Core Responsibilities
- Initialize a compress object for coefficient-based (transcoding) output via `jpeg_write_coefficients`
- Copy lossless-transcoding-safe parameters from a decompressor to a compressor via `jpeg_copy_critical_parameters`
- Select and wire up the minimal set of compression modules needed for transcoding (`transencode_master_selection`)
- Implement a specialized coefficient buffer controller that reads from pre-supplied virtual arrays instead of a pixel pipeline
- Generate on-the-fly dummy DCT padding blocks at image right/bottom edges during output

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_coef_controller` | struct | Private coefficient buffer controller; extends `jpeg_c_coef_controller` with iMCU row tracking, MCU counters, virtual array pointer, and dummy block workspace |
| `my_coef_ptr` | typedef | Pointer alias for `my_coef_controller` |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_write_coefficients
- **Signature:** `GLOBAL void jpeg_write_coefficients(j_compress_ptr cinfo, jvirt_barray_ptr *coef_arrays)`
- **Purpose:** Entry point to begin transcoding compression; sets up the compressor to write from existing DCT coefficient arrays rather than raw pixels.
- **Inputs:** `cinfo` — compress object (must be in `CSTATE_START`); `coef_arrays` — one virtual block array per component.
- **Outputs/Return:** void; modifies `cinfo->global_state` to `CSTATE_WRCOEFS`.
- **Side effects:** Resets error manager and destination module; calls `transencode_master_selection`; writes SOI marker.
- **Calls:** `jpeg_suppress_tables`, `cinfo->err->reset_error_mgr`, `cinfo->dest->init_destination`, `transencode_master_selection`.
- **Notes:** Caller must subsequently call `jpeg_finish_compress()` to flush data.

### jpeg_copy_critical_parameters
- **Signature:** `GLOBAL void jpeg_copy_critical_parameters(j_decompress_ptr srcinfo, j_compress_ptr dstinfo)`
- **Purpose:** Copies all parameters required for lossless transcoding (dimensions, colorspace, quantization tables, component sampling factors) from a decompressor to a compressor.
- **Inputs:** `srcinfo` — fully-read decompressor; `dstinfo` — fresh compressor in `CSTATE_START`.
- **Outputs/Return:** void; populates `dstinfo` fields.
- **Side effects:** Allocates quantization tables in `dstinfo` pool if not already present; calls `jpeg_set_defaults` and `jpeg_set_colorspace`.
- **Calls:** `jpeg_set_defaults`, `jpeg_set_colorspace`, `jpeg_alloc_quant_table`.
- **Notes:** Validates that per-component quantization table assignments match slot contents; errors if mismatched (`JERR_MISMATCHED_QUANT_TABLE`). Huffman table assignments are intentionally NOT copied.

### transencode_master_selection
- **Signature:** `LOCAL void transencode_master_selection(j_compress_ptr cinfo, jvirt_barray_ptr *coef_arrays)`
- **Purpose:** Replaces `jcinit.c`'s full-compressor initialization; wires only the modules needed for transcoding (master control, entropy encoder, coefficient controller, marker writer).
- **Inputs:** `cinfo`, `coef_arrays`.
- **Side effects:** Calls `jinit_c_master_control`, selects Huffman or progressive Huffman encoder, calls `transencode_coef_controller`, `jinit_marker_writer`, realizes virtual arrays, writes SOI header.
- **Calls:** `jinit_c_master_control`, `jinit_phuff_encoder` (conditional), `jinit_huff_encoder`, `transencode_coef_controller`, `jinit_marker_writer`, `cinfo->mem->realize_virt_arrays`, `cinfo->marker->write_file_header`.
- **Notes:** Arithmetic coding is explicitly unsupported (`JERR_ARITH_NOTIMPL`).

### compress_output
- **Signature:** `METHODDEF boolean compress_output(j_compress_ptr cinfo, JSAMPIMAGE input_buf)`
- **Purpose:** Per-iMCU-row output pump; reads DCT blocks from virtual arrays and feeds them to the entropy encoder.
- **Inputs:** `cinfo`; `input_buf` is ignored (NULL in transcoding path).
- **Outputs/Return:** `TRUE` if iMCU row completed; `FALSE` if entropy encoder suspended (output buffer full).
- **Side effects:** Updates `coef->iMCU_row_num`, `coef->mcu_ctr`, `coef->MCU_vert_offset`; calls entropy encoder per MCU.
- **Calls:** `cinfo->mem->access_virt_barray`, `cinfo->entropy->encode_mcu`, `start_iMCU_row`.
- **Notes:** Generates dummy padding blocks on-the-fly at right/bottom edges; DC value of dummy block propagated from preceding real block; AC entries pre-zeroed at allocation.

### transencode_coef_controller
- **Signature:** `LOCAL void transencode_coef_controller(j_compress_ptr cinfo, jvirt_barray_ptr *coef_arrays)`
- **Purpose:** Allocates and initializes the `my_coef_controller` object; installs `start_pass_coef` and `compress_output` as method pointers; pre-allocates and zeroes dummy block buffer.
- **Side effects:** Allocates from `JPOOL_IMAGE`; sets `cinfo->coef`.
- **Calls:** `cinfo->mem->alloc_small`, `cinfo->mem->alloc_large`, `jzero_far`.

### Notes (minor functions)
- `start_iMCU_row`: resets `mcu_ctr` and `MCU_vert_offset`; computes `MCU_rows_per_iMCU_row` accounting for interleaved vs. non-interleaved scans and bottom-of-image truncation.
- `start_pass_coef`: validates pass mode is `JBUF_CRANK_DEST`; resets row counters.

## Control Flow Notes
This file is active only during the **compression pass** of a transcode workflow (decompress → `jpeg_read_coefficients` → `jpeg_copy_critical_parameters` + `jpeg_write_coefficients` → `jpeg_finish_compress`). It does not participate in normal pixel-data compression init or in any render/frame loop. `compress_output` is driven by `jpeg_finish_compress` iterating iMCU rows.

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — all JPEG object types, method interfaces, constants
- **Defined elsewhere:** `jpeg_suppress_tables`, `jpeg_set_defaults`, `jpeg_set_colorspace`, `jpeg_alloc_quant_table`, `jinit_c_master_control`, `jinit_huff_encoder`, `jinit_phuff_encoder`, `jinit_marker_writer`, `jzero_far`
