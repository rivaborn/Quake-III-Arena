# code/jpeg-6/jidctflt.c

## File Purpose
Implements a floating-point inverse DCT (IDCT) with integrated dequantization for the IJG JPEG library. It converts an 8×8 block of quantized DCT coefficients back into pixel-domain sample values using the Arai, Agui, and Nakajima (AA&N) scaled DCT algorithm.

## Core Responsibilities
- Dequantize input JCOEF coefficients by multiplying against the component's float multiplier table
- Perform a separable 2-pass (column then row) 8-point floating-point IDCT
- Short-circuit columns where all AC terms are zero (DC-only optimization)
- Descale results by factor of 8 (2³) in the row pass
- Range-limit final values to valid `JSAMPLE` range via lookup table
- Write output samples into the caller-supplied output row buffer

## Key Types / Data Structures
None defined in this file; all types come from `jpeglib.h` and `jdct.h`.

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_idct_float
- **Signature:** `GLOBAL void jpeg_idct_float(j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- **Purpose:** Performs dequantization and 2D floating-point IDCT on one 8×8 DCT coefficient block, producing 8 rows of 8 output samples.
- **Inputs:**
  - `cinfo` — decompressor context; used to obtain the range-limit table via `IDCT_range_limit()`
  - `compptr` — component info; `compptr->dct_table` provides the `FLOAT_MULT_TYPE` dequantization multiplier array
  - `coef_block` — pointer to the 64-element array of quantized `JCOEF` input coefficients
  - `output_buf` — 2D array of output rows
  - `output_col` — column offset into each output row
- **Outputs/Return:** Void; writes 8×8 reconstructed pixel samples directly into `output_buf[0..7][output_col..output_col+7]`.
- **Side effects:** Writes to caller-owned output buffer; uses stack-allocated 64-element `FAST_FLOAT workspace[DCTSIZE2]` as intermediate buffer between passes. No heap allocation, no I/O.
- **Calls:**
  - `IDCT_range_limit(cinfo)` — macro resolving to `cinfo->sample_range_limit + CENTERJSAMPLE`
  - `DEQUANTIZE(coef, quantval)` — macro casting coef to `FAST_FLOAT` and multiplying by quantval
  - `DESCALE(x, n)` — macro for right-shift rounding (used with n=3 in row pass)
- **Notes:**
  - Guarded by `#ifdef DCT_FLOAT_SUPPORTED`; entire function is a no-op compile if disabled
  - Hardcoded to `DCTSIZE == 8`; compile error (deliberate syntax error) if `DCTSIZE != 8`
  - Pass 1 (columns): detects all-AC-zero columns and fills workspace with the single DC value, skipping butterfly math
  - Pass 2 (rows): does not attempt zero-AC optimization (float comparison cost vs. low occurrence rate)
  - AA&N constants used: √2 ≈ 1.414213562, 2·cos(π/8) ≈ 1.847759065, 2·(cos(π/8)−cos(3π/8)) ≈ 1.082392200, −2·(cos(π/8)+cos(3π/8)) ≈ −2.613125930

## Control Flow Notes
Called during the decompression pipeline's IDCT stage, invoked per DCT block by `jddctmgr.c`. It is a leaf operation within the per-MCU decode loop: coefficient entropy decoding → dequantize+IDCT (this file) → upsampling → color conversion → output. No init or shutdown involvement.

## External Dependencies
- `jinclude.h` — platform includes (`stdio.h`, `string.h`, etc.) and utility macros
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `FLOAT_MULT_TYPE`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `FAST_FLOAT`
- `jmorecfg.h` (via `jpeglib.h`) — `FAST_FLOAT`, `MULTIPLIER`, `CENTERJSAMPLE`, `MAXJSAMPLE`, `SHIFT_TEMPS`
- `compptr->dct_table` — populated externally by `jddctmgr.c` during decompressor startup
- `cinfo->sample_range_limit` — populated externally by `jdmaster.c` (`prepare_range_limit_table`)
