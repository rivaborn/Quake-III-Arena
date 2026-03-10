# code/jpeg-6/jdsample.c

## File Purpose
Implements the upsampling stage of the JPEG decompression pipeline. It expands chroma (and other subsampled) components back to full output resolution, optionally using bilinear ("fancy") interpolation or simple box-filter replication.

## Core Responsibilities
- Initialize the upsampler module and select per-component upsample methods during decompression setup
- Buffer one row group of upsampled data in `color_buf` before passing to color conversion
- Support multiple upsampling strategies: fullsize passthrough, no-op, integer box-filter, fast 2h1v/2h2v box, and fancy triangle-filter variants
- Track remaining image rows to handle images whose height is not a multiple of `max_v_samp_factor`
- Allocate intermediate color conversion buffers only for components that actually require rescaling

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_upsampler` | struct (private subobject) | Extends `jpeg_upsampler`; holds color conversion buffers, per-component method pointers, row counters, and pixel expansion factors |
| `upsample1_ptr` | typedef (function pointer) | Signature for a per-component upsample routine |

## Global / File-Static State
None.

## Key Functions / Methods

### start_pass_upsample
- **Signature:** `METHODDEF void start_pass_upsample(j_decompress_ptr cinfo)`
- **Purpose:** Resets the upsampler state at the start of each decompression pass.
- **Inputs:** `cinfo` — decompressor context
- **Outputs/Return:** void
- **Side effects:** Sets `next_row_out = max_v_samp_factor` (marks buffer empty); sets `rows_to_go = output_height`
- **Calls:** none
- **Notes:** Called via `upsample->pub.start_pass` vtable slot.

### sep_upsample
- **Signature:** `METHODDEF void sep_upsample(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION *in_row_group_ctr, JDIMENSION in_row_groups_avail, JSAMPARRAY output_buf, JDIMENSION *out_row_ctr, JDIMENSION out_rows_avail)`
- **Purpose:** Main upsampling driver: fills the intermediate `color_buf` one row group at a time, then color-converts and emits as many rows as possible.
- **Inputs:** Compressed component input rows, current row group counter, output scanline buffer and counters
- **Outputs/Return:** void; advances `*in_row_group_ctr` and `*out_row_ctr`
- **Side effects:** Writes to `output_buf`; updates `upsample->next_row_out`, `rows_to_go`; calls color converter
- **Calls:** Per-component `upsample->methods[ci]`, `cinfo->cconvert->color_convert`
- **Notes:** Advances `in_row_group_ctr` only after the full buffer has been consumed; clamped by `rows_to_go` to handle non-multiple image heights.

### fullsize_upsample
- **Signature:** `METHODDEF void fullsize_upsample(..., JSAMPARRAY input_data, JSAMPARRAY *output_data_ptr)`
- **Purpose:** Zero-copy passthrough for components already at full output size.
- **Side effects:** Sets `*output_data_ptr = input_data`

### noop_upsample
- **Signature:** `METHODDEF void noop_upsample(..., JSAMPARRAY *output_data_ptr)`
- **Purpose:** No-op for components not needed by color conversion; sets pointer to NULL.

### int_upsample
- **Signature:** `METHODDEF void int_upsample(...)`
- **Purpose:** Generic integer-ratio box-filter upsampling for arbitrary integer H/V expansion factors.
- **Side effects:** Writes expanded rows into `*output_data_ptr`; calls `jcopy_sample_rows` for vertical duplication.
- **Notes:** Uses cached `h_expand`/`v_expand` from `my_upsampler`; not optimized for speed.

### h2v1_upsample / h2v2_upsample
- **Purpose:** Fast 2:1 horizontal (and optionally 2:1 vertical) box-filter specializations; duplicate each input sample directly without interpolation.
- **Notes:** h2v2 calls `jcopy_sample_rows` to duplicate the first output row.

### h2v1_fancy_upsample
- **Purpose:** Triangle-filter (linear interpolation) 2:1 H, 1:1 V upsampling. Output pixels placed at 1/4 and 3/4 positions between input pixel centers using alternating bias to avoid systematic rounding error.

### h2v2_fancy_upsample
- **Purpose:** Triangle-filter 2:1 H × 2:1 V upsampling with 9/16–3/16–3/16–1/16 weighting across both axes. Requires adjacent input rows (context rows).
- **Notes:** Sets `upsample->pub.need_context_rows = TRUE` when selected.

### jinit_upsampler
- **Signature:** `GLOBAL void jinit_upsampler(j_decompress_ptr cinfo)`
- **Purpose:** Module init — allocates `my_upsampler`, wires vtable slots, selects per-component method, and allocates `color_buf` rows where needed.
- **Inputs:** `cinfo`
- **Side effects:** Allocates from `JPOOL_IMAGE`; sets `cinfo->upsample`; may set `need_context_rows = TRUE`; calls `ERREXIT` on unsupported sampling (CCIR601, fractional ratios)
- **Calls:** `cinfo->mem->alloc_small`, `cinfo->mem->alloc_sarray`, `jround_up`

## Control Flow Notes
Called during decompression init (`jinit_upsampler`). Each output pass invokes `start_pass_upsample`, then `sep_upsample` is driven per-row-group by `jdpostct.c` / `jdmainct.c` until all rows are emitted. The file does not participate in compression or rendering.

## External Dependencies
- `jinclude.h` — platform portability macros
- `jpeglib.h` / `jpegint.h` — JPEG decompressor structs, `jpeg_upsampler`, `jpeg_component_info`, sample typedefs
- `jcopy_sample_rows` — defined in `jutils.c`
- `jround_up` — defined in `jutils.c`
- `ERREXIT` — error macro from `jerror.h`
