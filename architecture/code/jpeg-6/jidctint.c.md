# code/jpeg-6/jidctint.c

## File Purpose
Implements the slow-but-accurate integer inverse DCT (IDCT) for the IJG JPEG library, performing combined dequantization and 2D IDCT on a single 8×8 DCT coefficient block. This is the `JDCT_ISLOW` variant, based on the Loeffler–Ligtenberg–Moschytz algorithm with 12 multiplies and 32 adds per 1-D pass.

## Core Responsibilities
- Dequantize 64 DCT coefficients using the component's quantization multiplier table
- Execute a two-pass separable 2D IDCT (column pass then row pass) on an 8×8 block
- Apply scaled fixed-point arithmetic with compile-time integer constants to avoid floating-point at runtime
- Short-circuit all-zero AC columns/rows for a common-case speedup
- Range-limit and clamp all 64 output samples into valid `JSAMPLE` (0–255) values
- Write the decoded 8×8 pixel block into the caller-supplied output scanline buffer

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ISLOW_MULT_TYPE` | typedef (from `jdct.h`) | Element type for the dequantization multiplier table (`MULTIPLIER`, typically `short` or `int`) |
| `INT32` | typedef | 32-bit signed integer used for all intermediate fixed-point arithmetic |
| `JCOEFPTR` | typedef | Pointer to input DCT coefficient block (`JCOEF FAR *`) |
| `JSAMPARRAY` | typedef | 2-D array of output pixel rows |

## Global / File-Static State

None.

## Key Functions / Methods

### jpeg_idct_islow
- **Signature:** `GLOBAL void jpeg_idct_islow(j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- **Purpose:** Perform dequantization + full-precision integer IDCT on one 8×8 DCT block, writing 8 rows × 8 samples into `output_buf`.
- **Inputs:**
  - `cinfo` — decompressor context; provides `sample_range_limit` table via `IDCT_range_limit()`
  - `compptr` — component info; `compptr->dct_table` points to the `ISLOW_MULT_TYPE[64]` multiplier table
  - `coef_block` — 64 quantized DCT coefficients in natural (raster) order
  - `output_buf` — array of output row pointers
  - `output_col` — horizontal offset within each output row
- **Outputs/Return:** `void`; writes decoded pixel samples directly into `output_buf[0..7][output_col..output_col+7]`
- **Side effects:** Writes to caller-supplied `output_buf`. Stack-allocates `workspace[64]` (256 bytes) as inter-pass buffer. No heap allocation, no I/O.
- **Calls:** `DEQUANTIZE`, `MULTIPLY` (macros expanding to integer multiply), `DESCALE` (macro for right-shift with rounding), `IDCT_range_limit` (macro accessing `cinfo->sample_range_limit`).
- **Notes:**
  - Pass 1 iterates over 8 columns; all-zero AC check fires ~50%+ of the time with typical images.
  - Pass 2 iterates over 8 rows; all-zero AC check (`NO_ZERO_ROW_TEST` guard) fires ~5–10% of the time.
  - Fixed-point precision: `CONST_BITS=13`, `PASS1_BITS=2` (8-bit samples). Final descale in pass 2 is `CONST_BITS+PASS1_BITS+3 = 18` bits.
  - Hardcoded for `DCTSIZE=8`; a deliberate syntax error enforces this at compile time.
  - Entire function is conditionally compiled under `#ifdef DCT_ISLOW_SUPPORTED`.

## Control Flow Notes

This file is part of the JPEG **decompression** pipeline. It is called per-block during the IDCT stage, invoked by `jddctmgr.c` through the `jpeg_inverse_dct` module's function pointer. It runs inside the decode loop — once per 8×8 MCU block component — and has no role in init or shutdown. It does not call into the broader engine; all communication is through its parameters.

## External Dependencies

- `jinclude.h` — platform includes, `MEMZERO`/`MEMCOPY`, `size_t`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `ISLOW_MULT_TYPE`, `DESCALE`, `FIX`, `MULTIPLY16C16`, `IDCT_range_limit`, `RANGE_MASK`, `CONST_SCALE`
- `jmorecfg.h` (via `jpeglib.h`) — `INT32`, `MULTIPLIER`, `BITS_IN_JSAMPLE`, `CENTERJSAMPLE`, `MAXJSAMPLE`, `SHIFT_TEMPS`, `RIGHT_SHIFT`
- **Defined elsewhere:** `IDCT_range_limit` table populated by `jdmaster.c:prepare_range_limit_table()`; multiplier table (`compptr->dct_table`) populated by `jddctmgr.c`
