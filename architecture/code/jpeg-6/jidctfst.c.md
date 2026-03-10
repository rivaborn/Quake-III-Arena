# code/jpeg-6/jidctfst.c

## File Purpose
Implements a fast, reduced-accuracy integer Inverse Discrete Cosine Transform (IDCT) for the IJG JPEG decompression library. It performs simultaneous dequantization and 8x8 block IDCT using the Arai, Agui & Nakajima (AA&N) scaled algorithm, trading precision for speed compared to the slow/accurate variant (`jidctint.c`).

## Core Responsibilities
- Dequantize 64 DCT coefficients using a precomputed multiplier table (`compptr->dct_table`)
- Execute a two-pass separable 1-D IDCT (columns first, then rows) on a single 8x8 block
- Short-circuit computation for columns/rows with all-zero AC terms (DC-only fast path)
- Scale and range-limit all 64 output pixels into valid `JSAMPLE` values (0–MAXJSAMPLE)
- Write one reconstructed 8x8 tile of pixel data into the caller-supplied output buffer

## Key Types / Data Structures
None defined in this file; relies entirely on types from included headers.

| Name | Kind | Purpose |
|---|---|---|
| `DCTELEM` | typedef (int or INT32) | Intermediate fixed-point DCT computation element |
| `IFAST_MULT_TYPE` | typedef (MULTIPLIER or INT32) | Dequantization multiplier table element type |
| `JSAMPLE` | typedef (unsigned char) | Final output pixel sample type |
| `JSAMPROW` / `JSAMPARRAY` | typedef (pointer) | Row/array of output pixel samples |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_idct_ifast
- **Signature:** `GLOBAL void jpeg_idct_ifast(j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- **Purpose:** Perform dequantization + fast integer IDCT on one 8×8 DCT coefficient block, producing 8 rows of 8 decoded pixels written into `output_buf`.
- **Inputs:**
  - `cinfo` — decompressor context; used to retrieve `sample_range_limit` table
  - `compptr` — component info; `compptr->dct_table` provides per-coefficient scale/quantization multipliers
  - `coef_block` — 64-element array of quantized DCT coefficients in natural (non-zigzag) order
  - `output_buf` — array of output row pointers; destination for decoded pixel rows
  - `output_col` — column offset within each output row to start writing
- **Outputs/Return:** `void`; writes decoded pixel values directly to `output_buf[0..7][output_col..output_col+7]`
- **Side effects:** Writes to caller-provided output buffer; uses a 64-element `workspace[DCTSIZE2]` stack array as scratch between passes. No heap allocation, no I/O, no global state modified.
- **Calls:**
  - `IDCT_range_limit(cinfo)` — macro expanding to `cinfo->sample_range_limit + CENTERJSAMPLE`
  - `DEQUANTIZE`, `MULTIPLY`, `DESCALE`/`RIGHT_SHIFT`, `IDESCALE`, `IRIGHT_SHIFT` — all local macros
- **Notes:**
  - Entire function is conditionally compiled under `#ifdef DCT_IFAST_SUPPORTED`
  - Hard-coded to `DCTSIZE == 8`; a deliberate syntax error fires if that assumption is violated
  - **Pass 1 (columns):** iterates 8 columns; if all AC coefficients are zero, broadcasts dequantized DC value to all 8 workspace slots and skips butterfly math
  - **Pass 2 (rows):** iterates 8 rows from workspace; similar DC-only fast path gated by `#ifndef NO_ZERO_ROW_TEST`
  - Uses only 8 fractional bits for multiplicative constants (`CONST_BITS=8`) to reduce shift cost; this causes more rounding error at high-quality quantization tables
  - With `USE_ACCURATE_ROUNDING` undefined, `DESCALE` omits the rounding add, producing slightly wrong results ~50% of the time for a small further speed gain
  - Fixed-point constants `FIX_1_082392200` (277), `FIX_1_414213562` (362), `FIX_1_847759065` (473), `FIX_2_613125930` (669) are pre-calculated to avoid runtime float ops on compilers that cannot fold `FIX()` at compile time

## Control Flow Notes
Called per-block during JPEG decompression by the IDCT manager (`jddctmgr.c`) as a function pointer registered during decompressor initialization. Invoked once per 8×8 MCU block during the output phase; not part of init or shutdown. There is no frame/tick loop here — each call is a fully self-contained transformation of one block.

## External Dependencies
- `jinclude.h` — platform portability, `MEMZERO`/`MEMCOPY`, system headers
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `DCTELEM`, `IFAST_MULT_TYPE`, `IFAST_SCALE_BITS`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `FIX`
- `jmorecfg.h` (via jpeglib.h) — `BITS_IN_JSAMPLE`, `MULTIPLIER`, `INT32`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- **Defined elsewhere:** `IDCT_range_limit` result table populated by `jdmaster.c:prepare_range_limit_table`; `compptr->dct_table` populated by `jddctmgr.c`; `DCT_IFAST_SUPPORTED` guard defined in `jconfig.h`
