# code/jpeg-6/jdcoefct.c

## File Purpose
Implements the coefficient buffer controller for JPEG decompression, sitting between entropy decoding and inverse-DCT stages. Manages both single-pass (streaming) and multi-pass (buffered-image, progressive) decompression modes, including optional interblock smoothing for progressive scans.

## Core Responsibilities
- Buffer MCU coefficient blocks received from the entropy decoder
- Drive the inverse-DCT (IDCT) transform per component block
- Coordinate input and output passes in multi-scan/buffered-image mode
- Implement JPEG K.8 interblock smoothing for progressive scans
- Initialize and wire up the `jpeg_d_coef_controller` vtable on startup

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_coef_controller` | struct | Private extension of `jpeg_d_coef_controller`; holds MCU row counters, MCU block buffer, optional virtual image arrays, and smoothing latch |
| `my_coef_ptr` | typedef | Pointer alias for `my_coef_controller` |

## Global / File-Static State

None.

## Key Functions / Methods

### `start_iMCU_row`
- Signature: `LOCAL void start_iMCU_row(j_decompress_ptr cinfo)`
- Purpose: Resets per-iMCU-row counters (`MCU_ctr`, `MCU_vert_offset`, `MCU_rows_per_iMCU_row`) at the start of each input row.
- Inputs: `cinfo` — decompressor state
- Outputs/Return: void; mutates `coef` fields
- Side effects: Writes `coef->MCU_rows_per_iMCU_row`, `MCU_ctr`, `MCU_vert_offset`
- Calls: none
- Notes: Handles boundary case at last iMCU row using `last_row_height`

### `start_input_pass`
- Signature: `METHODDEF void start_input_pass(j_decompress_ptr cinfo)`
- Purpose: Entry point to begin a new input processing pass; resets `input_iMCU_row` to 0 and calls `start_iMCU_row`.
- Inputs: `cinfo`
- Outputs/Return: void
- Side effects: Sets `cinfo->input_iMCU_row = 0`
- Calls: `start_iMCU_row`

### `start_output_pass`
- Signature: `METHODDEF void start_output_pass(j_decompress_ptr cinfo)`
- Purpose: Entry point to begin a new output pass; optionally selects `decompress_smooth_data` over `decompress_data` if smoothing is applicable.
- Inputs: `cinfo`
- Outputs/Return: void
- Side effects: May reassign `coef->pub.decompress_data` function pointer; resets `cinfo->output_iMCU_row = 0`
- Calls: `smoothing_ok` (conditional)

### `decompress_onepass`
- Signature: `METHODDEF int decompress_onepass(j_decompress_ptr cinfo, JSAMPIMAGE output_buf)`
- Purpose: Single-pass decompression; decodes one iMCU row directly from entropy decoder to output samples without storing the full image.
- Inputs: `cinfo`, `output_buf` — per-component sample planes
- Outputs/Return: `JPEG_ROW_COMPLETED`, `JPEG_SCAN_COMPLETED`, or `JPEG_SUSPENDED`
- Side effects: Writes to `output_buf`; advances `cinfo->input_iMCU_row`, `cinfo->output_iMCU_row`; calls `finish_input_pass` at end of scan
- Calls: `jzero_far`, `cinfo->entropy->decode_mcu`, IDCT via `inverse_DCT` function pointer, `start_iMCU_row`, `cinfo->inputctl->finish_input_pass`
- Notes: Skips dummy blocks at image edges; skips unneeded components (`component_needed`)

### `dummy_consume_data`
- Signature: `METHODDEF int dummy_consume_data(j_decompress_ptr cinfo)`
- Purpose: No-op consume_data stub for single-pass mode where input/output are locked in step.
- Outputs/Return: Always `JPEG_SUSPENDED`

### `consume_data` *(D_MULTISCAN_FILES_SUPPORTED)*
- Signature: `METHODDEF int consume_data(j_decompress_ptr cinfo)`
- Purpose: Reads one iMCU row from the entropy decoder into the full-image virtual block arrays.
- Inputs: `cinfo`
- Outputs/Return: `JPEG_ROW_COMPLETED`, `JPEG_SCAN_COMPLETED`, or `JPEG_SUSPENDED`
- Side effects: Writes decoded DCT coefficients into `coef->whole_image` virtual arrays
- Calls: `cinfo->mem->access_virt_barray`, `cinfo->entropy->decode_mcu`, `start_iMCU_row`, `cinfo->inputctl->finish_input_pass`

### `decompress_data` *(D_MULTISCAN_FILES_SUPPORTED)*
- Signature: `METHODDEF int decompress_data(j_decompress_ptr cinfo, JSAMPIMAGE output_buf)`
- Purpose: Multi-pass output path; reads one iMCU row from virtual arrays and applies IDCT to produce output samples.
- Inputs: `cinfo`, `output_buf`
- Outputs/Return: `JPEG_ROW_COMPLETED` or `JPEG_SCAN_COMPLETED`
- Side effects: May call `consume_input` to synchronize input ahead of output
- Calls: `cinfo->inputctl->consume_input`, `cinfo->mem->access_virt_barray`, IDCT via `inverse_DCT`

### `smoothing_ok` *(BLOCK_SMOOTHING_SUPPORTED)*
- Signature: `LOCAL boolean smoothing_ok(j_decompress_ptr cinfo)`
- Purpose: Validates preconditions for interblock smoothing and latches current `coef_bits` accuracy values.
- Inputs: `cinfo`
- Outputs/Return: `TRUE` if smoothing is applicable and useful
- Side effects: Allocates `coef->coef_bits_latch` on first call; writes latched values
- Calls: `cinfo->mem->alloc_small`
- Notes: Returns `FALSE` for non-progressive files or missing quantization tables

### `decompress_smooth_data` *(BLOCK_SMOOTHING_SUPPORTED)*
- Signature: `METHODDEF int decompress_smooth_data(j_decompress_ptr cinfo, JSAMPIMAGE output_buf)`
- Purpose: Variant of `decompress_data` that estimates missing AC coefficients (AC01, AC10, AC20, AC11, AC02) from neighboring DC values per JPEG Annex K.8 before applying IDCT.
- Inputs: `cinfo`, `output_buf`
- Outputs/Return: `JPEG_ROW_COMPLETED` or `JPEG_SCAN_COMPLETED`
- Side effects: Writes modified coefficients into a local `workspace` (not back to virtual arrays); advances `output_iMCU_row`
- Calls: `cinfo->inputctl->consume_input`, `cinfo->mem->access_virt_barray`, `jcopy_block_row`, IDCT via `inverse_DCT`
- Notes: Uses a 3×3 sliding window of DC values (DC1–DC9) for neighbor estimation

### `jinit_d_coef_controller`
- Signature: `GLOBAL void jinit_d_coef_controller(j_decompress_ptr cinfo, boolean need_full_buffer)`
- Purpose: Allocates and initializes the coefficient controller, setting up either full virtual arrays (multi-pass) or a single-MCU workspace (single-pass), and wires the vtable.
- Inputs: `cinfo`, `need_full_buffer`
- Outputs/Return: void; installs `cinfo->coef`
- Side effects: Allocates memory via `cinfo->mem`; assigns `consume_data`/`decompress_data` function pointers
- Calls: `cinfo->mem->alloc_small`, `cinfo->mem->alloc_large`, `cinfo->mem->request_virt_barray`
- Notes: Errors (`ERREXIT`) if `need_full_buffer` but `D_MULTISCAN_FILES_SUPPORTED` is not compiled in

## Control Flow Notes
Called once during decompressor initialization via `jinit_d_coef_controller`. Subsequently, `start_input_pass` / `start_output_pass` are called at scan boundaries by the decompressor master. Per-row, `consume_data` (input side) and `decompress_data` / `decompress_onepass` (output side) are invoked by the main decompressor loop to pipeline entropy decoding → IDCT → sample output.

## External Dependencies
- `jinclude.h`, `jpeglib.h` (via `jpegint.h`, `jerror.h`)
- **Defined elsewhere:** `jzero_far`, `jcopy_block_row`, `jround_up`; all IDCT implementations (`inverse_DCT_method_ptr`); entropy decoder (`decode_mcu`); memory manager (`access_virt_barray`, `request_virt_barray`, `alloc_small`, `alloc_large`); input controller (`consume_input`, `finish_input_pass`)
