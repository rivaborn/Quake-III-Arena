# code/jpeg-6/jdmerge.c

## File Purpose
Implements a merged upsampling and YCbCr-to-RGB color conversion pass for JPEG decompression. By combining chroma upsampling and colorspace conversion into a single loop, it avoids redundant per-pixel multiplications for the shared chroma terms, yielding a significant throughput improvement for the common 2h1v and 2h2v chroma subsampling cases.

## Core Responsibilities
- Build precomputed integer lookup tables for YCbCr→RGB channel contributions from Cb and Cr
- Provide a `start_pass` routine that resets per-pass state (spare row, row counter)
- Dispatch upsampling via `merged_2v_upsample` (2:1 vertical) or `merged_1v_upsample` (1:1 vertical)
- Implement `h2v1_merged_upsample`: process one luma row, 2:1 horizontal chroma replication, emit one output row
- Implement `h2v2_merged_upsample`: process two luma rows, 2:1 horizontal and 2:1 vertical chroma replication, emit two output rows
- Manage a spare row buffer for the 2v case when the caller supplies only a single-row output buffer, and discard the dummy last row on odd-height images
- Register itself as `cinfo->upsample` during module initialization

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `my_upsampler` | struct (private subobject of `jpeg_upsampler`) | Extends the public upsampler vtable with per-instance state: conversion tables, spare row buffer, row bookkeeping, and dispatch function pointer |
| `my_upsample_ptr` | typedef (pointer) | Convenience alias for `my_upsampler *` |

## Global / File-Static State
None.

## Key Functions / Methods

### build_ycc_rgb_table
- **Signature:** `LOCAL void build_ycc_rgb_table(j_decompress_ptr cinfo)`
- **Purpose:** Allocates and populates four integer lookup tables (one per channel contribution: Cr→R, Cb→B, Cr→G, Cb→G) covering the full 0..MAXJSAMPLE range.
- **Inputs:** `cinfo` — active decompression context
- **Outputs/Return:** void; tables stored in `upsample->Cr_r_tab`, `Cb_b_tab`, `Cr_g_tab`, `Cb_g_tab`
- **Side effects:** Allocates four heap arrays from `JPOOL_IMAGE` via `cinfo->mem->alloc_small`
- **Calls:** `cinfo->mem->alloc_small`
- **Notes:** Uses fixed-point arithmetic (`SCALEBITS=16`, `FIX`, `RIGHT_SHIFT`). ITU-R BT.601 coefficients: 1.402, 1.772, 0.71414, 0.34414. The Cb→G table absorbs the `ONE_HALF` rounding bias so inner loops avoid an extra add.

### start_pass_merged_upsample
- **Signature:** `METHODDEF void start_pass_merged_upsample(j_decompress_ptr cinfo)`
- **Purpose:** Resets per-pass state at the start of each decompression output pass.
- **Inputs:** `cinfo`
- **Outputs/Return:** void
- **Side effects:** Sets `spare_full = FALSE` and `rows_to_go = cinfo->output_height`
- **Calls:** None

### merged_2v_upsample
- **Signature:** `METHODDEF void merged_2v_upsample(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION *in_row_group_ctr, JDIMENSION in_row_groups_avail, JSAMPARRAY output_buf, JDIMENSION *out_row_ctr, JDIMENSION out_rows_avail)`
- **Purpose:** Control wrapper for 2:1 vertical sampling; handles spare-row buffering so the inner conversion function always receives two valid output row pointers.
- **Inputs:** Input sample planes, mutable row counters, output buffer
- **Outputs/Return:** void; advances `*out_row_ctr` and `*in_row_group_ctr`
- **Side effects:** May write to `upsample->spare_row`; toggles `spare_full`; decrements `rows_to_go`
- **Calls:** `jcopy_sample_rows` (spare drain path), `upsample->upmethod` (h2v2_merged_upsample)
- **Notes:** When the caller cannot accept two rows, the second row is silently written to the spare buffer and returned on the next call.

### merged_1v_upsample
- **Signature:** `METHODDEF void merged_1v_upsample(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION *in_row_group_ctr, JDIMENSION in_row_groups_avail, JSAMPARRAY output_buf, JDIMENSION *out_row_ctr, JDIMENSION out_rows_avail)`
- **Purpose:** Trivial control wrapper for 1:1 vertical sampling; directly invokes the inner method and advances counters.
- **Side effects:** Increments both `*out_row_ctr` and `*in_row_group_ctr`
- **Calls:** `upsample->upmethod` (h2v1_merged_upsample)

### h2v1_merged_upsample
- **Signature:** `METHODDEF void h2v1_merged_upsample(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION in_row_group_ctr, JSAMPARRAY output_buf)`
- **Purpose:** Inner loop for 2:1 horizontal, 1:1 vertical: reads one Cb and one Cr sample, computes chroma offsets once, applies them to two consecutive Y samples, emits two RGB pixels.
- **Inputs:** YCbCr input planes indexed by `in_row_group_ctr`, single output row pointer
- **Side effects:** Writes directly to caller-supplied `output_buf[0]`
- **Notes:** Handles odd output widths with a trailing single-pixel case. Uses `range_limit` table for clamping.

### h2v2_merged_upsample
- **Signature:** `METHODDEF void h2v2_merged_upsample(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION in_row_group_ctr, JSAMPARRAY output_buf)`
- **Purpose:** Inner loop for 2:1 horizontal, 2:1 vertical: one Cb/Cr sample pair is shared by four Y samples across two luma rows.
- **Side effects:** Writes to `output_buf[0]` and `output_buf[1]`
- **Notes:** Accesses luma rows at `in_row_group_ctr*2` and `in_row_group_ctr*2+1`.

### jinit_merged_upsampler
- **Signature:** `GLOBAL void jinit_merged_upsampler(j_decompress_ptr cinfo)`
- **Purpose:** Module initializer; allocates the `my_upsampler` object, wires vtable pointers, sets dispatch based on `max_v_samp_factor`, optionally allocates the spare row, and builds the YCC→RGB tables.
- **Side effects:** Allocates memory from `JPOOL_IMAGE`; sets `cinfo->upsample`
- **Calls:** `cinfo->mem->alloc_small`, `cinfo->mem->alloc_large`, `build_ycc_rgb_table`
- **Notes:** Called only when `use_merged_upsample()` in `jdmaster.c` returns TRUE; no capability re-checking is performed here.

## Control Flow Notes
This module is activated during the JPEG decompression output phase. `jinit_merged_upsampler` is called at startup. Each call to `jpeg_read_scanlines` ultimately drives `cinfo->upsample->upsample` — either `merged_2v_upsample` or `merged_1v_upsample` — which dispatches to the appropriate inner `upmethod`. The module is entirely passive between passes and is re-armed by `start_pass_merged_upsample`.

## External Dependencies
- `jinclude.h` — platform portability, `SIZEOF`, `MEMCOPY`
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_upsampler`, `JSAMPIMAGE`, `JDIMENSION`, `INT32`, `JSAMPLE`, `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE`, `JPOOL_IMAGE`, `SHIFT_TEMPS`, `RIGHT_SHIFT`, `FIX`
- `jcopy_sample_rows` — defined in `jutils.c` (used in spare-row drain path)
- `use_merged_upsample` — defined in `jdmaster.c` (controls whether this module is selected)
- `cinfo->sample_range_limit` — populated by `jdmaster.c` startup
