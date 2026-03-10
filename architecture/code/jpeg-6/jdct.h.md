# code/jpeg-6/jdct.h

## File Purpose
Private shared header for the JPEG DCT/IDCT subsystem within the Independent JPEG Group (IJG) library. It defines types, macros, and external declarations used by both the forward DCT (compression) and inverse DCT (decompression) modules and their per-algorithm implementation files.

## Core Responsibilities
- Define `DCTELEM` as the working integer type for forward DCT buffers (width depends on sample bit depth)
- Declare function pointer typedefs for forward DCT method dispatch (`forward_DCT_method_ptr`, `float_DCT_method_ptr`)
- Define per-algorithm multiplier types for IDCT dequantization tables (`ISLOW_MULT_TYPE`, `IFAST_MULT_TYPE`, `FLOAT_MULT_TYPE`)
- Provide the range-limiting macro `IDCT_range_limit` and `RANGE_MASK` for safe output clamping
- Declare all forward and inverse DCT routine entry points as `EXTERN`
- Supply fixed-point arithmetic macros (`FIX`, `DESCALE`, `MULTIPLY16C16`, `MULTIPLY16V16`) used across DCT implementation files
- Provide short-name aliases for linkers that cannot handle long external symbols

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `DCTELEM` | typedef | Element type for forward DCT work buffer; `int` for 8-bit samples, `INT32` for 12-bit |
| `forward_DCT_method_ptr` | typedef (function pointer) | Dispatch type for integer forward DCT implementations |
| `float_DCT_method_ptr` | typedef (function pointer) | Dispatch type for floating-point forward DCT implementation |
| `ISLOW_MULT_TYPE` | typedef | Dequantization multiplier type for slow-integer IDCT (`MULTIPLIER`) |
| `IFAST_MULT_TYPE` | typedef | Dequantization multiplier type for fast-integer IDCT; 16-bit for 8-bit samples, `INT32` for 12-bit |
| `FLOAT_MULT_TYPE` | typedef | Dequantization multiplier type for floating-point IDCT (`FAST_FLOAT`) |

## Global / File-Static State
None.

## Key Functions / Methods
This is a header file; no function bodies are defined here. All entries below are `EXTERN` declarations.

### jpeg_fdct_islow / jpeg_fdct_ifast / jpeg_fdct_float
- Signature: `void (DCTELEM *data)` / `void (FAST_FLOAT *data)`
- Purpose: Perform an in-place 8×8 forward DCT on a coefficient work buffer using slow-integer, fast-integer, or floating-point algorithm respectively.
- Inputs: Pointer to 64-element work array (signed, centered at 0).
- Outputs/Return: In-place; outputs are scaled up by factor of 8.
- Side effects: Modifies the caller-supplied buffer only.
- Notes: Defined in `jfdctint.c`, `jfdctfst.c`, `jfdctflt.c`.

### jpeg_idct_islow / jpeg_idct_ifast / jpeg_idct_float
- Signature: `void (j_decompress_ptr cinfo, jpeg_component_info *compptr, JCOEFPTR coef_block, JSAMPARRAY output_buf, JDIMENSION output_col)`
- Purpose: Dequantize and inverse-DCT one 8×8 block, writing reconstructed samples into the output sample array at the specified column offset.
- Inputs: Decompressor state, component info (carries `dct_table` pointer), input coefficient block, output buffer, column offset.
- Outputs/Return: Writes `DCT_scaled_size × DCT_scaled_size` samples into `output_buf`.
- Side effects: Reads `compptr->dct_table`; range-limits results via `sample_range_limit` table in `cinfo`.
- Notes: Defined in `jidctint.c`, `jidctfst.c`, `jidctflt.c`.

### jpeg_idct_4x4 / jpeg_idct_2x2 / jpeg_idct_1x1
- Signature: Same as IDCT variants above.
- Purpose: Scaled-down IDCT variants producing 4×4, 2×2, or 1×1 output blocks for partial-resolution decoding.
- Notes: Defined in `jidctred.c`.

## Control Flow Notes
This header is included only by DCT manager files (`jcdctmgr.c`, `jddctmgr.c`) and the individual algorithm implementation files. It is not part of the public JPEG API. During compression, `jcdctmgr.c` calls a `forward_DCT_method_ptr` pointing to one of the `jpeg_fdct_*` routines per 8×8 block. During decompression, `jddctmgr.c` dispatches through `inverse_DCT_method_ptr` (declared in `jpegint.h`) to one of the `jpeg_idct_*` routines.

## External Dependencies
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JMETHOD`, `JPP`, `EXTERN`, `inverse_DCT_method_ptr`
- `jmorecfg.h` — `BITS_IN_JSAMPLE`, `MULTIPLIER`, `FAST_FLOAT`, `INT32`, `INT16`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- `RIGHT_SHIFT` — defined elsewhere (platform/compiler-specific macro, typically in `jconfig.h` or `jpegint.h`)
- `SHORTxSHORT_32`, `SHORTxLCONST_32` — optional compile-time flags defined by platform configuration; govern `MULTIPLY16C16` implementation
