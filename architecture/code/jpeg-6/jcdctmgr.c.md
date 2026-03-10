# code/jpeg-6/jcdctmgr.c

## File Purpose
Manages the forward DCT (Discrete Cosine Transform) pipeline for JPEG compression. It selects the appropriate DCT algorithm at initialization, precomputes scaled quantization divisor tables per component, and drives the encode-time DCT-and-quantize step for each 8×8 sample block.

## Core Responsibilities
- Allocate and initialize the `my_fdct_controller` subobject and wire it into `cinfo->fdct`
- Select the active DCT routine (`jpeg_fdct_islow`, `jpeg_fdct_ifast`, or `jpeg_fdct_float`) based on `cinfo->dct_method`
- Precompute per-quantization-table divisor arrays (scaled and reordered from zigzag) during `start_pass`
- Load 8×8 pixel blocks into a workspace with unsigned-to-signed bias removal
- Invoke the chosen DCT routine in-place on the workspace
- Quantize/descale the 64 DCT coefficients and write them to the output coefficient block array

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `my_fdct_controller` | struct | Private controller extending `jpeg_forward_dct`; holds function pointers to the active DCT routine and precomputed divisor tables for up to 4 quantization tables |
| `my_fdct_ptr` | typedef | Pointer alias for `my_fdct_controller *` |
| `forward_DCT_method_ptr` | typedef (function ptr) | Signature for integer DCT routines operating on `DCTELEM[]` |
| `float_DCT_method_ptr` | typedef (function ptr) | Signature for floating-point DCT routines operating on `FAST_FLOAT[]` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `aanscales[DCTSIZE2]` | `static const INT16[64]` | static (local to `start_pass_fdctmgr`, IFAST branch) | Precomputed AA&N scale factors × 2^14 for fast integer quantization |
| `aanscalefactor[DCTSIZE]` | `static const double[8]` | static (local to `start_pass_fdctmgr`, float branch) | Per-row/col AA&N cosine scale factors for floating-point quantization |

## Key Functions / Methods

### start_pass_fdctmgr
- **Signature:** `METHODDEF void start_pass_fdctmgr(j_compress_ptr cinfo)`
- **Purpose:** Called before each compression pass; validates that each component's referenced quantization table exists and builds the scaled divisor array for the selected DCT method.
- **Inputs:** `cinfo` — compressor state including `comp_info`, `quant_tbl_ptrs`, `dct_method`
- **Outputs/Return:** void; populates `fdct->divisors[]` (integer methods) or `fdct->float_divisors[]` (float method) via JPOOL_IMAGE allocation
- **Side effects:** Allocates memory through `cinfo->mem->alloc_small`; calls `ERREXIT1` / `ERREXIT` on invalid table index or unsupported method
- **Calls:** `cinfo->mem->alloc_small`, `ERREXIT1`, `ERREXIT`, `DESCALE`, `MULTIPLY16V16`
- **Notes:** Divisors are reordered from JPEG zigzag order to natural (raster) order; the float path stores reciprocals (1/divisor) to replace division with multiplication in the hot loop

### forward_DCT_float
- **Signature:** `METHODDEF void forward_DCT_float(j_compress_ptr cinfo, jpeg_component_info *compptr, JSAMPARRAY sample_data, JBLOCKROW coef_blocks, JDIMENSION start_row, JDIMENSION start_col, JDIMENSION num_blocks)`
- **Purpose:** Hot-path: processes `num_blocks` consecutive 8×8 sample blocks — loads samples with center-bias, runs the float DCT, multiplies by precomputed reciprocal divisors, rounds to nearest integer, and stores `JCOEF` output.
- **Inputs:** `sample_data` (pixel rows), `start_row`/`start_col` (block origin), `num_blocks` (count), `compptr->quant_tbl_no` (selects divisor table)
- **Outputs/Return:** void; writes quantized DCT coefficients into `coef_blocks[]`
- **Side effects:** None beyond writing output; uses stack-allocated `FAST_FLOAT workspace[64]`
- **Calls:** `fdct->do_float_dct` (function pointer → `jpeg_fdct_float`), `GETJSAMPLE`
- **Notes:** Rounding trick `(int)(temp + 16384.5) - 16384` handles both positive and negative values portably without a branch; inner loop manually unrolled for DCTSIZE==8

### jinit_forward_dct
- **Signature:** `GLOBAL void jinit_forward_dct(j_compress_ptr cinfo)`
- **Purpose:** One-time initialization: allocates the `my_fdct_controller`, sets method pointers, assigns the correct DCT function pair, and clears all divisor table pointers to NULL.
- **Inputs:** `cinfo` — must have `dct_method` already set
- **Outputs/Return:** void; sets `cinfo->fdct`
- **Side effects:** Allocates permanent image-pool memory; calls `ERREXIT` for unsupported `dct_method`
- **Calls:** `cinfo->mem->alloc_small`, `ERREXIT`
- **Notes:** The integer `forward_DCT` function is compiled out (`#if 0`) in this source; only `forward_DCT_float` is active at runtime in this build

## Control Flow Notes
- **Init:** `jinit_forward_dct` is called once during compression startup.
- **Pass start:** `start_pass_fdctmgr` (via `fdct->pub.start_pass`) is invoked at the beginning of each compression pass to rebuild divisor tables if needed.
- **Per-block (encode loop):** `fdct->pub.forward_DCT` (→ `forward_DCT_float`) is called by the coefficient controller for every MCU block row, forming the innermost encode-path bottleneck.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, `JBLOCKROW`, `JSAMPARRAY`, `JCOEF`, `NUM_QUANT_TBLS`, `DCTSIZE2`, `JPOOL_IMAGE`
- `jdct.h` — `DCTELEM`, `forward_DCT_method_ptr`, `float_DCT_method_ptr`, `FAST_FLOAT`, fixed-point macros (`DESCALE`, `MULTIPLY16V16`, `SHIFT_TEMPS`)
- **Defined elsewhere:** `jpeg_fdct_islow` (`jfdctint.c`), `jpeg_fdct_ifast` (`jfdctfst.c`), `jpeg_fdct_float` (`jfdctflt.c`), `jpeg_zigzag_order` (IJG internal table), `ERREXIT`/`ERREXIT1` (error handler macros from `jerror.h`)
