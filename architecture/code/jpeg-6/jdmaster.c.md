# code/jpeg-6/jdmaster.c

## File Purpose
Master control module for the IJG JPEG decompressor. It selects which decompression sub-modules to activate, configures multi-pass quantization, and drives the per-pass setup/teardown lifecycle called by `jdapi.c`.

## Core Responsibilities
- Compute output image dimensions and DCT scaling factors (`jpeg_calc_output_dimensions`)
- Build the sample range-limit lookup table for fast pixel clamping (`prepare_range_limit_table`)
- Select and initialize all active decompressor sub-modules (IDCT, entropy decoder, upsampler, color converter, quantizer, buffer controllers)
- Manage per-output-pass start/finish sequencing and dummy-pass logic for 2-pass color quantization
- Expose `jinit_master_decompress` as the library entry point that boots the master object

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_decomp_master` | struct | Private extension of `jpeg_decomp_master`; holds pass counter, merged-upsample flag, and saved pointers to both quantizer instances |
| `my_master_ptr` | typedef (pointer) | Typed pointer to `my_decomp_master` for internal casts |

## Global / File-Static State

None.

## Key Functions / Methods

### use_merged_upsample
- **Signature:** `LOCAL boolean use_merged_upsample(j_decompress_ptr cinfo)`
- **Purpose:** Decides whether the merged upsample+color-convert path (`jdmerge.c`) can be used instead of separate modules.
- **Inputs:** `cinfo` — decompression context with color space, component, and sampling parameters set.
- **Outputs/Return:** `TRUE` if all conditions are met, `FALSE` otherwise.
- **Side effects:** None.
- **Calls:** None (pure predicate).
- **Notes:** Guards are: no fancy upsampling, no CCIR601, YCbCr→RGB only, 2h1v or 2h2v sampling, uniform `DCT_scaled_size` across all three components. Compiled out entirely if `UPSAMPLE_MERGING_SUPPORTED` is not defined.

---

### jpeg_calc_output_dimensions
- **Signature:** `GLOBAL void jpeg_calc_output_dimensions(j_decompress_ptr cinfo)`
- **Purpose:** Computes `output_width/height`, per-component `DCT_scaled_size` and `downsampled_width/height`, `out_color_components`, `output_components`, and `rec_outbuf_height`.
- **Inputs:** `cinfo` with image dimensions, scale ratio, and color space parameters.
- **Outputs/Return:** Void; writes results directly into `cinfo` fields.
- **Side effects:** Modifies multiple `cinfo` output descriptor fields.
- **Calls:** `use_merged_upsample`, `jdiv_round_up`, `ERREXIT1`.
- **Notes:** May be called by the application before `jpeg_start_decompress`. Must be idempotent (safe to call twice). When `IDCT_SCALING_SUPPORTED` is absent, output dimensions equal input dimensions and per-component scaling is a no-op.

---

### prepare_range_limit_table
- **Signature:** `LOCAL void prepare_range_limit_table(j_decompress_ptr cinfo)`
- **Purpose:** Allocates and fills `cinfo->sample_range_limit`, a lookup table for fast clamping of IDCT output to [0, MAXJSAMPLE] with wraparound safety masking.
- **Inputs:** `cinfo`.
- **Outputs/Return:** Void; sets `cinfo->sample_range_limit`.
- **Side effects:** Allocates `JPOOL_IMAGE` memory via `cinfo->mem->alloc_small`.
- **Calls:** `MEMZERO`, `MEMCOPY`.
- **Notes:** Table pointer is offset so negative indices are valid. Post-IDCT section maps corrupt out-of-range values safely via bitmasking (`x & MASK`). Table layout documented extensively in the source comments.

---

### master_selection
- **Signature:** `LOCAL void master_selection(j_decompress_ptr cinfo)`
- **Purpose:** Orchestrates full module selection: calls dimension/table setup then conditionally calls `jinit_*` for every active sub-module based on `cinfo` parameters.
- **Inputs:** `cinfo` after `jpeg_read_header`.
- **Outputs/Return:** Void.
- **Side effects:** Initializes all decompressor sub-objects; calls `realize_virt_arrays` and `start_input_pass`; may adjust progress monitor counters.
- **Calls:** `jpeg_calc_output_dimensions`, `prepare_range_limit_table`, `jinit_1pass_quantizer`, `jinit_2pass_quantizer`, `jinit_merged_upsampler`, `jinit_color_deconverter`, `jinit_upsampler`, `jinit_d_post_controller`, `jinit_inverse_dct`, `jinit_phuff_decoder` / `jinit_huff_decoder`, `jinit_d_coef_controller`, `jinit_d_main_controller`, `realize_virt_arrays`, `start_input_pass`, `ERREXIT`.
- **Notes:** Arithmetic coding always errors out (`JERR_ARITH_NOTIMPL`). Width overflow check guards against scanline row size exceeding `JDIMENSION`.

---

### prepare_for_output_pass
- **Signature:** `METHODDEF void prepare_for_output_pass(j_decompress_ptr cinfo)`
- **Purpose:** Called at the start of each output pass; selects active quantizer, calls `start_pass` on all active modules, sets `is_dummy_pass`, and updates progress monitor.
- **Inputs:** `cinfo`; `master->pub.is_dummy_pass` from prior pass.
- **Outputs/Return:** Void.
- **Side effects:** Dispatches `start_pass` to `idct`, `coef`, `cconvert`, `upsample`, `cquantize`, `post`, and `main` sub-objects.
- **Calls:** Sub-object `start_pass` / `start_output_pass` function pointers.
- **Notes:** When entering the final pass of 2-pass quantization (`is_dummy_pass` was TRUE), flips to `FALSE` and reconfigures pipeline for `JBUF_CRANK_DEST`.

---

### finish_output_pass
- **Signature:** `METHODDEF void finish_output_pass(j_decompress_ptr cinfo)`
- **Purpose:** Called at the end of each output pass; flushes quantizer if active and increments `pass_number`.
- **Inputs:** `cinfo`.
- **Outputs/Return:** Void.
- **Side effects:** Calls `cquantize->finish_pass`; increments `master->pass_number`.
- **Calls:** `cquantize->finish_pass`.

---

### jpeg_new_colormap
- **Signature:** `GLOBAL void jpeg_new_colormap(j_decompress_ptr cinfo)`
- **Purpose:** Switches to a new externally-supplied colormap between output passes in buffered-image mode.
- **Inputs:** `cinfo` with updated `colormap`.
- **Outputs/Return:** Void.
- **Side effects:** Replaces `cinfo->cquantize` with the saved 2-pass quantizer; calls `new_color_map` on it.
- **Calls:** `cquantize->new_color_map`, `ERREXIT1`, `ERREXIT`.
- **Notes:** Only valid in `DSTATE_BUFIMAGE` state; guarded by `D_MULTISCAN_FILES_SUPPORTED`.

---

### jinit_master_decompress
- **Signature:** `GLOBAL void jinit_master_decompress(j_decompress_ptr cinfo)`
- **Purpose:** Library entry point; allocates and wires up the `my_decomp_master` object and triggers `master_selection`.
- **Inputs:** `cinfo` immediately before `jpeg_start_decompress`.
- **Outputs/Return:** Void; sets `cinfo->master`.
- **Side effects:** Allocates `JPOOL_IMAGE` memory; installs `prepare_for_output_pass` and `finish_output_pass` method pointers.
- **Calls:** `alloc_small`, `master_selection`.

## Control Flow Notes
This file sits at the **init** phase of decompression. `jinit_master_decompress` is called once from `jpeg_start_decompress` (in `jdapi.c`). Thereafter, `prepare_for_output_pass` / `finish_output_pass` are invoked once per output pass (including dummy passes for 2-pass quantization) from the same API layer. The file has no per-row or per-frame tick; all logic is setup/teardown only.

## External Dependencies
- **Includes:** `jinclude.h`, `jpeglib.h` (pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`)
- **Defined elsewhere:** `jdiv_round_up` (jutils.c), `jinit_1pass_quantizer` (jquant1.c), `jinit_2pass_quantizer` (jquant2.c), `jinit_merged_upsampler` (jdmerge.c), `jinit_color_deconverter` (jdcolor.c), `jinit_upsampler` (jdsample.c), `jinit_d_post_controller` (jdpostct.c), `jinit_inverse_dct` (jddctmgr.c), `jinit_phuff_decoder` (jdphuff.c), `jinit_huff_decoder` (jdhuff.c), `jinit_d_coef_controller` (jdcoefct.c), `jinit_d_main_controller` (jdmainct.c)
