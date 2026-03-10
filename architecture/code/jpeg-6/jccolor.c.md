# code/jpeg-6/jccolor.c

## File Purpose
Implements input colorspace conversion for the IJG JPEG compressor. It transforms application-supplied pixel data (RGB, CMYK, grayscale, YCbCr, YCCK) into the JPEG internal colorspace before encoding. This is the compression-side counterpart to `jdcolor.c`.

## Core Responsibilities
- Allocate and initialize lookup tables for fixed-point RGB→YCbCr conversion
- Convert interleaved RGB input rows to planar YCbCr output (most common path)
- Convert RGB rows to grayscale (Y-only)
- Convert CMYK rows to YCCK (inverts CMY, passes K through)
- Pass through grayscale or multi-component data unchanged (`null_convert`, `grayscale_convert`)
- Select and wire the correct conversion function pointer pair (`start_pass` + `color_convert`) during module initialization

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_color_converter` | struct | Private extension of `jpeg_color_converter`; holds the `rgb_ycc_tab` lookup table pointer |
| `my_cconvert_ptr` | typedef | Convenience pointer to `my_color_converter` |

## Global / File-Static State
None.

## Key Functions / Methods

### rgb_ycc_start
- **Signature:** `METHODDEF void rgb_ycc_start(j_compress_ptr cinfo)`
- **Purpose:** Allocates and populates the 8-section fixed-point lookup table used by RGB→YCbCr, RGB→grayscale, and CMYK→YCCK converters.
- **Inputs:** `cinfo` — active compression context
- **Outputs/Return:** void; writes `cconvert->rgb_ycc_tab`
- **Side effects:** Allocates `TABLE_SIZE * sizeof(INT32)` bytes from `JPOOL_IMAGE`
- **Calls:** `cinfo->mem->alloc_small`
- **Notes:** Table entries fold in CCIR 601-1 coefficients, `CENTERJSAMPLE` offset, and rounding bias so the inner loops need only table lookups and a right-shift. `R_CR_OFF` aliases `B_CB_OFF` because those two partial products are numerically identical.

### rgb_ycc_convert
- **Signature:** `METHODDEF void rgb_ycc_convert(j_compress_ptr cinfo, JSAMPARRAY input_buf, JSAMPIMAGE output_buf, JDIMENSION output_row, int num_rows)`
- **Purpose:** Core RGB→YCbCr row converter; reformats interleaved RGB pixels into three separate planar component buffers.
- **Inputs:** interleaved `input_buf` rows; `output_buf` planar destination; `output_row` start index; `num_rows` count
- **Outputs/Return:** void; fills `output_buf[0..2][output_row..]`
- **Side effects:** None beyond output buffer writes
- **Calls:** `GETJSAMPLE` (macro)
- **Notes:** No range-limiting needed because the fixed-point math guarantees outputs stay in `[0, MAXJSAMPLE]`. Inner loop advances `inptr` by `RGB_PIXELSIZE` per pixel.

### rgb_gray_convert
- **Signature:** `METHODDEF void rgb_gray_convert(j_compress_ptr cinfo, JSAMPARRAY input_buf, JSAMPIMAGE output_buf, JDIMENSION output_row, int num_rows)`
- **Purpose:** Extracts only the Y (luminance) component from RGB; reuses the Y portion of `rgb_ycc_tab`.
- **Inputs/Outputs:** Same pattern as `rgb_ycc_convert`; writes only `output_buf[0]`
- **Side effects:** None
- **Notes:** Requires `rgb_ycc_start` to have been called first.

### cmyk_ycck_convert
- **Signature:** `METHODDEF void cmyk_ycck_convert(j_compress_ptr cinfo, JSAMPARRAY input_buf, JSAMPIMAGE output_buf, JDIMENSION output_row, int num_rows)`
- **Purpose:** Adobe-style CMYK→YCCK: inverts C/M/Y to pseudo-RGB, applies RGB→YCbCr math, passes K channel unchanged.
- **Outputs/Return:** void; fills `output_buf[0..3]`
- **Notes:** K channel copied directly without `GETJSAMPLE` since no range extension is needed.

### grayscale_convert
- **Signature:** `METHODDEF void grayscale_convert(j_compress_ptr cinfo, JSAMPARRAY input_buf, JSAMPIMAGE output_buf, JDIMENSION output_row, int num_rows)`
- **Purpose:** Copies the first byte of each interleaved pixel into the single output plane; handles arbitrary `input_components` stride.
- **Notes:** No lookup table required; `null_method` is used for `start_pass`.

### null_convert
- **Signature:** `METHODDEF void null_convert(j_compress_ptr cinfo, JSAMPARRAY input_buf, JSAMPIMAGE output_buf, JDIMENSION output_row, int num_rows)`
- **Purpose:** De-interleaves N-component input into N separate output planes with no value transformation.

### jinit_color_converter
- **Signature:** `GLOBAL void jinit_color_converter(j_compress_ptr cinfo)`
- **Purpose:** Module entry point. Allocates `my_color_converter`, validates `input_components` vs. `in_color_space`, then selects and assigns `start_pass` + `color_convert` function pointers based on the `in_color_space` → `jpeg_color_space` mapping.
- **Inputs:** `cinfo` with `in_color_space`, `jpeg_color_space`, `input_components`, `num_components` already set
- **Side effects:** Allocates converter object; sets `cinfo->cconvert`; calls `ERREXIT` on unsupported combinations
- **Calls:** `cinfo->mem->alloc_small`, `ERREXIT`

### Notes
- `null_method` is a trivial no-op assigned to `start_pass` when no table initialization is required.

## Control Flow Notes
`jinit_color_converter` is called during compressor initialization (startup). Per-row conversion happens during the compression pass: the preprocessing controller calls `cconvert->pub.start_pass` once at pass start, then `cconvert->pub.color_convert` for each batch of input rows — both are function pointers set at init time.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `JSAMPLE*` types, `jpeg_color_converter`, `J_COLOR_SPACE` enum, `JPOOL_IMAGE`, `ERREXIT`, `METHODDEF`, `GLOBAL`
- `jmorecfg.h` (via jpeglib) — `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE`, `INT32`
- `alloc_small` — defined in the JPEG memory manager (`jmemmgr.c`), called through `cinfo->mem`
