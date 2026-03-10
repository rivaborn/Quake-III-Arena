# code/jpeg-6/jddctmgr.c

## File Purpose
Manages the inverse DCT (IDCT) subsystem for JPEG decompression. It selects the appropriate IDCT implementation per component based on the requested DCT method and output scaling, and builds the corresponding dequantization multiplier tables used by the IDCT routines.

## Core Responsibilities
- Select the correct IDCT function pointer per component based on `DCT_scaled_size` and `dct_method`
- Convert raw zigzag-ordered quantization tables into method-specific multiplier tables (ISLOW, IFAST, FLOAT)
- Pre-zero multiplier tables so uninitialized components produce neutral gray output
- Cache the current IDCT method per component to avoid redundant table rebuilds
- Initialize and register the IDCT controller subobject with the decompressor

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_idct_controller` | struct | Private IDCT controller extending `jpeg_inverse_dct`; stores per-component method codes in `cur_method[]` |
| `my_idct_ptr` | typedef | Pointer alias for `my_idct_controller` |
| `multiplier_table` | union | Variant storage large enough to hold ISLOW, IFAST, or FLOAT multiplier arrays (64 elements each) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `aanscales` | `static const INT16[64]` | local to `start_pass` (IFAST branch) | Precomputed AA&N scale factors scaled by 14 bits for IFAST multiplier setup |
| `aanscalefactor` | `static const double[8]` | local to `start_pass` (FLOAT branch) | Per-row/col AA&N scale factors for FLOAT multiplier setup |

## Key Functions / Methods

### start_pass
- **Signature:** `METHODDEF void start_pass(j_decompress_ptr cinfo)`
- **Purpose:** Called at the start of each output pass to assign IDCT function pointers and rebuild multiplier tables for each component.
- **Inputs:** `cinfo` — decompressor instance with `comp_info`, `dct_method`, `idct`
- **Outputs/Return:** `void`; modifies `idct->pub.inverse_DCT[ci]` and `compptr->dct_table` for each component
- **Side effects:** Writes into per-component `dct_table` memory; calls `ERREXIT`/`ERREXIT1` on invalid configuration
- **Calls:** `jpeg_idct_1x1`, `jpeg_idct_2x2`, `jpeg_idct_4x4`, `jpeg_idct_islow`, `jpeg_idct_ifast`, `jpeg_idct_float`, `DESCALE`, `MULTIPLY16V16`, `ERREXIT`, `ERREXIT1`
- **Notes:** Skips rebuild if `!component_needed` or `cur_method[ci] == method`; leaves table zeroed if `quant_table == NULL` (handles buffered-image mode before data arrives); zigzag reordering via `jpeg_zigzag_order[]`

### jinit_inverse_dct
- **Signature:** `GLOBAL void jinit_inverse_dct(j_decompress_ptr cinfo)`
- **Purpose:** Allocates and initializes the IDCT controller, allocates zeroed multiplier tables for each component, and registers `start_pass` as the output-pass setup hook.
- **Inputs:** `cinfo` — decompressor instance
- **Outputs/Return:** `void`; sets `cinfo->idct`
- **Side effects:** Allocates `JPOOL_IMAGE` memory via `cinfo->mem->alloc_small`; zero-initializes all multiplier tables; sets `cur_method[ci] = -1`
- **Calls:** `cinfo->mem->alloc_small`, `MEMZERO`
- **Notes:** Must be called once during decompressor initialization before any output pass

## Control Flow Notes
`jinit_inverse_dct` is called during decompressor startup (init phase). `start_pass` is called once per output pass before IDCT processing begins; it is invoked via the `idct->pub.start_pass` function pointer registered here. No code in this file executes per-block during the actual IDCT step — that is handled by the selected `inverse_DCT[ci]` function pointers.

## External Dependencies
- `jinclude.h` — platform includes, `MEMZERO`, `SIZEOF`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, `J_DCT_METHOD` enum, `JPOOL_IMAGE`, `MAX_COMPONENTS`, `DCTSIZE`, `DCTSIZE2`
- `jdct.h` — `ISLOW_MULT_TYPE`, `IFAST_MULT_TYPE`, `FLOAT_MULT_TYPE`, `IFAST_SCALE_BITS`, `DESCALE`, `MULTIPLY16V16`, `SHIFT_TEMPS`, IDCT extern declarations
- **Defined elsewhere:** `jpeg_idct_islow`, `jpeg_idct_ifast`, `jpeg_idct_float`, `jpeg_idct_4x4`, `jpeg_idct_2x2`, `jpeg_idct_1x1` (individual IDCT implementation files); `jpeg_zigzag_order` (defined in `jutils.c`); `jpeg_inverse_dct` struct (defined in `jpegint.h`)
