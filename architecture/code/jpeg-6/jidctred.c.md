# code/jpeg-6/jidctred.c

## File Purpose
Implements reduced-size inverse DCT (IDCT) routines for JPEG decompression, producing 4x4, 2x2, or 1x1 pixel output from an 8x8 DCT coefficient block. These are used when downscaled image output is requested, avoiding a full 8x8 IDCT followed by downsampling.

## Core Responsibilities
- Dequantize DCT coefficients using the component's quantization table
- Perform a two-pass (column then row) reduced IDCT using simplified LL&M butterfly arithmetic
- Clamp output samples to valid range via a pre-built range-limit lookup table
- Write reduced-size pixel rows directly into the output sample buffer
- Short-circuit all-zero AC coefficient cases for performance

## Key Types / Data Structures

None (all types imported from `jpeglib.h` / `jdct.h`).

| Name | Kind | Purpose |
|------|------|---------|
| `ISLOW_MULT_TYPE` | typedef | Multiplier type for dequantization table entries |
| `INT32` | typedef | 32-bit signed integer used for fixed-point arithmetic |
| `JSAMPLE` | typedef | Single output pixel sample (8 or 12 bits) |

## Global / File-Static State

None.

## Key Functions / Methods

### jpeg_idct_4x4
- **Signature:** `GLOBAL void jpeg_idct_4x4(j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- **Purpose:** Dequantizes and inverse-DCTs one 8x8 coefficient block into a 4x4 pixel output block.
- **Inputs:** `cinfo` — decompressor state (provides range-limit table); `compptr` — component info (provides `dct_table`); `coef_block` — 64 quantized DCT coefficients; `output_buf`/`output_col` — destination sample array and column offset.
- **Outputs/Return:** Writes 4 rows × 4 pixels into `output_buf[0..3][output_col..output_col+3]`. Returns void.
- **Side effects:** Writes to caller-supplied output buffer; uses stack-allocated `workspace[DCTSIZE*4]`.
- **Calls:** `DEQUANTIZE`, `MULTIPLY`, `DESCALE`, `IDCT_range_limit` (all macros).
- **Notes:** Column 4 (index `DCTSIZE-4`) is skipped in pass 1 as the 4x4 reduction does not need it. Zero-AC fast path in both passes. Fixed-point constants assume `CONST_BITS == 13`.

### jpeg_idct_2x2
- **Signature:** `GLOBAL void jpeg_idct_2x2(j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- **Purpose:** Dequantizes and inverse-DCTs one 8x8 coefficient block into a 2x2 pixel output block.
- **Inputs:** Same pattern as `jpeg_idct_4x4`.
- **Outputs/Return:** Writes 2 rows × 2 pixels into `output_buf[0..1][output_col..output_col+1]`. Returns void.
- **Side effects:** Writes to caller-supplied output buffer; uses stack-allocated `workspace[DCTSIZE*2]`.
- **Calls:** `DEQUANTIZE`, `MULTIPLY`, `DESCALE`, `IDCT_range_limit` (all macros).
- **Notes:** Columns 2, 4, 6 skipped in pass 1. Only odd-row coefficients (1, 3, 5, 7) contribute to the odd part; even AC terms beyond DC are discarded by the 2x2 reduction.

### jpeg_idct_1x1
- **Signature:** `GLOBAL void jpeg_idct_1x1(j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- **Purpose:** Trivially produces a single pixel from a DCT block by taking DC coefficient / 8.
- **Inputs:** Same pattern; only `coef_block[0]` and `quantptr[0]` are used.
- **Outputs/Return:** Writes one pixel to `output_buf[0][output_col]`. Returns void.
- **Side effects:** None beyond the single pixel write.
- **Calls:** `DEQUANTIZE`, `DESCALE`, `IDCT_range_limit` (macros).
- **Notes:** No AC terms used; no workspace needed.

## Control Flow Notes

These routines are plugged into `jpeg_component_info::dct_table` dispatch (via `jddctmgr.c`) at decompression startup when `DCT_scaled_size` is set to 4, 2, or 1 respectively. They are called once per DCT block during the IDCT pass of the decompression pipeline (inside the coefficient-to-spatial conversion stage). The entire file is guarded by `#ifdef IDCT_SCALING_SUPPORTED`.

## External Dependencies

- `jinclude.h` — platform portability macros (`MEMZERO`, etc.)
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `ISLOW_MULT_TYPE`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `FIX`, `MULTIPLY16C16`, DCT size/precision constants
- `DCTSIZE`, `BITS_IN_JSAMPLE`, `INT32`, `MAXJSAMPLE`, `CENTERJSAMPLE` — defined elsewhere in the JPEG library configuration headers
- `RIGHT_SHIFT`, `SHIFT_TEMPS` — platform-specific shift helpers defined elsewhere
