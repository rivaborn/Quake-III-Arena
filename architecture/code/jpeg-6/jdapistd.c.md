# code/jpeg-6/jdapistd.c

## File Purpose
Implements the standard public API for the JPEG decompression pipeline, covering the full-decompression path from `jpeg_start_decompress` through scanline reading to buffered-image mode control. It is intentionally separated from `jdapimin.c` so that transcoder-only builds do not pull in the full decompressor.

## Core Responsibilities
- Initialize and drive the decompressor through its state machine (`DSTATE_*` transitions)
- Absorb multi-scan input into the coefficient buffer during startup
- Handle dummy output passes required by two-pass quantization
- Provide scanline-at-a-time output via `jpeg_read_scanlines`
- Provide raw iMCU-row output via `jpeg_read_raw_data`
- Manage buffered-image mode via `jpeg_start_output` / `jpeg_finish_output`

## Key Types / Data Structures
None defined here; all types come from `jpeglib.h`.

| Name | Kind | Purpose |
|---|---|---|
| `j_decompress_ptr` | typedef (pointer to struct) | Handle to the decompression instance; all functions operate on it |
| `JSAMPARRAY` | typedef | 2-D array of pixel samples passed to `jpeg_read_scanlines` |
| `JSAMPIMAGE` | typedef | 3-D sample array (per-component) used for raw data output |
| `JDIMENSION` | typedef | Unsigned integer for image dimensions and row counts |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_start_decompress
- **Signature:** `GLOBAL boolean jpeg_start_decompress(j_decompress_ptr cinfo)`
- **Purpose:** Top-level entry point to begin decompression after `jpeg_read_header`. Initializes the master decompressor, optionally absorbs all scans into the coefficient buffer (progressive/multi-scan), then delegates to `output_pass_setup`.
- **Inputs:** `cinfo` — decompression instance in `DSTATE_READY`, `DSTATE_PRELOAD`, or `DSTATE_PRESCAN`
- **Outputs/Return:** `TRUE` when ready to read scanlines; `FALSE` if suspended (suspending data source)
- **Side effects:** Mutates `cinfo->global_state`; calls `jinit_master_decompress`; may consume input via `cinfo->inputctl->consume_input`; invokes progress monitor callback
- **Calls:** `jinit_master_decompress`, `cinfo->inputctl->consume_input`, `cinfo->progress->progress_monitor`, `output_pass_setup`, `ERREXIT1`
- **Notes:** `D_MULTISCAN_FILES_SUPPORTED` guards the multi-scan absorption loop; buffered-image mode exits early after master init

### output_pass_setup
- **Signature:** `LOCAL boolean output_pass_setup(j_decompress_ptr cinfo)`
- **Purpose:** Internal helper that prepares for an output pass, running any required dummy passes (two-pass quantization) and setting the final `global_state` to `DSTATE_SCANNING` or `DSTATE_RAW_OK`.
- **Inputs:** `cinfo` — must be in `DSTATE_PRESCAN` on re-entry, or any other state on first call
- **Outputs/Return:** `TRUE` when output is ready; `FALSE` if suspended mid-dummy-pass
- **Side effects:** Calls `prepare_for_output_pass` and `finish_output_pass` on master; resets `output_scanline`; mutates `global_state`
- **Calls:** `cinfo->master->prepare_for_output_pass`, `cinfo->main->process_data`, `cinfo->master->finish_output_pass`, `cinfo->progress->progress_monitor`, `ERREXIT`
- **Notes:** `QUANT_2PASS_SUPPORTED` guards the dummy-pass loop; suspension mid-dummy-pass leaves state as `DSTATE_PRESCAN` for safe re-entry

### jpeg_read_scanlines
- **Signature:** `GLOBAL JDIMENSION jpeg_read_scanlines(j_decompress_ptr cinfo, JSAMPARRAY scanlines, JDIMENSION max_lines)`
- **Purpose:** Read up to `max_lines` decompressed scanlines into caller-supplied buffer.
- **Inputs:** `cinfo` in `DSTATE_SCANNING`; `scanlines` — output row buffer; `max_lines` — maximum rows to read
- **Outputs/Return:** Number of scanlines actually written (may be less than `max_lines`)
- **Side effects:** Advances `cinfo->output_scanline`; invokes progress monitor
- **Calls:** `cinfo->main->process_data`, `cinfo->progress->progress_monitor`, `ERREXIT1`, `WARNMS`
- **Notes:** Returns 0 with a warning if called past end of image

### jpeg_read_raw_data
- **Signature:** `GLOBAL JDIMENSION jpeg_read_raw_data(j_decompress_ptr cinfo, JSAMPIMAGE data, JDIMENSION max_lines)`
- **Purpose:** Read exactly one iMCU row of raw downsampled data directly into a caller-supplied per-component buffer.
- **Inputs:** `cinfo` in `DSTATE_RAW_OK`; `data` — per-component sample image; `max_lines` — must be ≥ `lines_per_iMCU_row`
- **Outputs/Return:** Number of scanlines produced (0 if suspended, else `lines_per_iMCU_row`)
- **Side effects:** Advances `cinfo->output_scanline`
- **Calls:** `cinfo->coef->decompress_data`, `cinfo->progress->progress_monitor`, `ERREXIT`, `WARNMS`
- **Notes:** Validates buffer is large enough for one full iMCU row before proceeding

### jpeg_start_output
- **Signature:** `GLOBAL boolean jpeg_start_output(j_decompress_ptr cinfo, int scan_number)`
- **Purpose:** Begin one output pass in buffered-image mode, targeting a specific scan number.
- **Inputs:** `cinfo` in `DSTATE_BUFIMAGE` or `DSTATE_PRESCAN`; `scan_number` — desired scan (clamped to valid range)
- **Outputs/Return:** `TRUE` when ready; `FALSE` if suspended
- **Side effects:** Sets `cinfo->output_scan_number`; delegates to `output_pass_setup`
- **Calls:** `output_pass_setup`, `ERREXIT1`
- **Notes:** Guarded by `D_MULTISCAN_FILES_SUPPORTED`

### jpeg_finish_output
- **Signature:** `GLOBAL boolean jpeg_finish_output(j_decompress_ptr cinfo)`
- **Purpose:** Finalize a buffered-image output pass and advance the input controller until the next scan or EOI.
- **Inputs:** `cinfo` in `DSTATE_SCANNING`, `DSTATE_RAW_OK`, or `DSTATE_BUFPOST`
- **Outputs/Return:** `TRUE` on completion; `FALSE` if suspended waiting for input
- **Side effects:** Calls `finish_output_pass`; advances `global_state` to `DSTATE_BUFPOST` then `DSTATE_BUFIMAGE`; may consume input markers
- **Calls:** `cinfo->master->finish_output_pass`, `cinfo->inputctl->consume_input`, `ERREXIT1`
- **Notes:** Guarded by `D_MULTISCAN_FILES_SUPPORTED`; does not require full pass completion before calling

## Control Flow Notes
This file sits entirely in the **decompression output** phase. `jpeg_start_decompress` is called once after `jpeg_read_header` completes. For normal images, control flows: `jpeg_start_decompress` → `output_pass_setup` → caller loop calling `jpeg_read_scanlines` per frame → `jpeg_finish_decompress` (in `jdapimin.c`). For buffered-image mode, the outer loop is `jpeg_start_output` / `jpeg_read_scanlines` / `jpeg_finish_output` repeated per scan. There is no per-frame or render-loop hook here; the caller drives iteration entirely.

## External Dependencies
- `jinclude.h` — platform portability macros, system headers
- `jpeglib.h` — all public JPEG types and struct definitions; pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`
- `jinit_master_decompress` — defined in `jdmaster.c` (external)
- `cinfo->master->prepare_for_output_pass`, `finish_output_pass`, `is_dummy_pass` — implemented in `jdmaster.c`
- `cinfo->main->process_data` — implemented in `jdmainct.c`
- `cinfo->coef->decompress_data` — implemented in `jdcoefct.c`
- `cinfo->inputctl->consume_input`, `has_multiple_scans`, `eoi_reached` — implemented in `jdinput.c`
- `ERREXIT`, `ERREXIT1`, `WARNMS` — error macros resolving through `jerror.h` / `jdapimin.c`
