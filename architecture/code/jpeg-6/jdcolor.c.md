# code/jpeg-6/jdcolor.c

## File Purpose
Implements output colorspace conversion for the IJG JPEG decompressor. It converts decoded JPEG component planes (YCbCr, YCCK, grayscale, CMYK) into the application's requested output colorspace (RGB, CMYK, grayscale, or pass-through).

## Core Responsibilities
- Build lookup tables for fixed-point YCbCr→RGB conversion coefficients
- Convert YCbCr component planes to interleaved RGB pixel rows
- Convert YCCK component planes to interleaved CMYK pixel rows
- Pass through grayscale (Y-only) data unchanged
- Perform null (same-colorspace) plane-to-interleaved reformatting
- Initialize and wire the `jpeg_color_deconverter` subobject into `cinfo`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_color_deconverter` | struct | Extends `jpeg_color_deconverter` with four precomputed YCC→RGB lookup table pointers |
| `my_cconvert_ptr` | typedef | Pointer alias for `my_color_deconverter*` |

## Global / File-Static State
None.

## Key Functions / Methods

### build_ycc_rgb_table
- **Signature:** `LOCAL void build_ycc_rgb_table(j_decompress_ptr cinfo)`
- **Purpose:** Precomputes four per-sample lookup tables for fixed-point YCbCr→RGB conversion, eliminating per-pixel multiplications in the inner loop.
- **Inputs:** `cinfo` — decompress context; reads `cinfo->mem` for allocation.
- **Outputs/Return:** void; populates `cconvert->Cr_r_tab`, `Cb_b_tab`, `Cr_g_tab`, `Cb_g_tab`.
- **Side effects:** Allocates four arrays of size `MAXJSAMPLE+1` from `JPOOL_IMAGE`.
- **Calls:** `cinfo->mem->alloc_small`
- **Notes:** Uses `SCALEBITS=16` fixed-point; Cb/Cr G-channel tables are left scaled (combined before rounding in the inner loop). `ONE_HALF` is pre-added to `Cb_g_tab` to absorb the rounding step.

---

### ycc_rgb_convert
- **Signature:** `METHODDEF void ycc_rgb_convert(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION input_row, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Converts rows of planar YCbCr samples to interleaved RGB pixels using the precomputed tables.
- **Inputs:** Three input planes (Y, Cb, Cr); `num_rows` rows starting at `input_row`; `output_buf` receives interleaved RGB.
- **Outputs/Return:** void; writes `RGB_PIXELSIZE` bytes per pixel into `output_buf`.
- **Side effects:** Reads `cinfo->sample_range_limit` for clamping; no allocation.
- **Calls:** `GETJSAMPLE`, `RIGHT_SHIFT` (macros), `range_limit` table indexing.
- **Notes:** Range-limiting is mandatory to handle DCT quantization noise that can push values out of range.

---

### null_convert
- **Signature:** `METHODDEF void null_convert(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION input_row, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Reformats planar (non-interleaved) samples to interleaved layout without any colorspace conversion.
- **Inputs:** `num_components` input planes; `num_rows` rows.
- **Outputs/Return:** void; writes interleaved pixels to `output_buf[0]`.
- **Side effects:** None.
- **Notes:** Used for RGB→RGB (when `RGB_PIXELSIZE==3`) and CMYK→CMYK pass-throughs.

---

### grayscale_convert
- **Signature:** `METHODDEF void grayscale_convert(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION input_row, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Copies luminance plane (component 0) directly to output; also handles YCbCr→grayscale by ignoring Cb/Cr.
- **Calls:** `jcopy_sample_rows` (defined elsewhere).

---

### ycck_cmyk_convert
- **Signature:** `METHODDEF void ycck_cmyk_convert(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION input_row, JSAMPARRAY output_buf, int num_rows)`
- **Purpose:** Converts YCCK to CMYK: applies YCbCr→RGB inversion (C=1-R, M=1-G, Y=1-B) on the first three channels and passes K unchanged.
- **Side effects:** Requires `build_ycc_rgb_table` to have been called.
- **Notes:** Outputs 4-byte CMYK pixels; uses `MAXJSAMPLE - value` to invert RGB to CMY.

---

### jinit_color_deconverter
- **Signature:** `GLOBAL void jinit_color_deconverter(j_decompress_ptr cinfo)`
- **Purpose:** Module init entry point — allocates `my_color_deconverter`, validates component counts against JPEG colorspace, sets `color_convert` function pointer and `out_color_components`, and optionally calls `build_ycc_rgb_table`.
- **Inputs:** `cinfo` with `jpeg_color_space`, `out_color_space`, `num_components` already set.
- **Outputs/Return:** void; installs `cinfo->cconvert`.
- **Side effects:** Allocates subobject; may allocate YCC tables; sets `cinfo->output_components`; may clear `component_needed` flags on unused components.
- **Calls:** `build_ycc_rgb_table`, `ERREXIT`.

---

### start_pass_dcolor
- **Notes:** No-op `start_pass` method; required by the `jpeg_color_deconverter` interface.

## Control Flow Notes
Called once during decompressor startup (`jinit_color_deconverter`). Per-frame, `color_convert` is invoked by the post-processing controller for each row group emitted by the upsampler, converting component planes to interleaved output before quantization or delivery to the application.

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMCOPY`
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_color_deconverter`, `JSAMPIMAGE`, `JSAMPARRAY`, `JDIMENSION`, `J_COLOR_SPACE` enum, `JPOOL_IMAGE`, `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE/PIXELSIZE`
- `jcopy_sample_rows` — defined in `jutils.c`
- `RIGHT_SHIFT`, `SHIFT_TEMPS`, `FIX` — macros from `jpegint.h`/`jmorecfg.h`
- `cinfo->mem->alloc_small` — memory manager defined in `jmemmgr.c`
- `cinfo->sample_range_limit` — populated by `jpeg_start_decompress` in `jdmaster.c`
