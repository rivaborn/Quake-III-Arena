# code/jpeg-6/jcsample.c

## File Purpose
Implements the downsampling module for the IJG JPEG compressor. It reduces the spatial resolution of color components (chroma subsampling) from the input image resolution down to the component's coded resolution before DCT processing.

## Core Responsibilities
- Provide per-component downsampling method dispatch via `sep_downsample`
- Implement box-filter downsampling for arbitrary integer ratios (`int_downsample`)
- Implement optimized 1:1 passthrough (`fullsize_downsample`)
- Implement optimized 2h1v and 2h2v downsampling with alternating-bias dithering
- Implement smoothed variants of 2h2v and fullsize downsampling (conditional on `INPUT_SMOOTHING_SUPPORTED`)
- Handle horizontal edge padding via `expand_right_edge`
- Select and wire up the appropriate per-component method pointer during init

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `downsample1_ptr` | typedef (function pointer) | Signature for a single-component downsample routine |
| `my_downsampler` | struct | Private subobject extending `jpeg_downsampler`; holds per-component method pointer array |
| `my_downsample_ptr` | typedef | Pointer alias for `my_downsampler` |

## Global / File-Static State
None.

## Key Functions / Methods

### expand_right_edge
- **Signature:** `LOCAL void expand_right_edge(JSAMPARRAY image_data, int num_rows, JDIMENSION input_cols, JDIMENSION output_cols)`
- **Purpose:** Pads each row rightward by replicating the last valid pixel, filling up to `output_cols` so downsampling loops don't need boundary checks.
- **Inputs:** Row array, row count, valid input width, desired padded width.
- **Outputs/Return:** Modifies `image_data` rows in place.
- **Side effects:** Writes past `input_cols` into the allocated buffer margin.
- **Calls:** None.
- **Notes:** Assumes the source buffer was allocated wide enough to accommodate the padding.

---

### sep_downsample
- **Signature:** `METHODDEF void sep_downsample(j_compress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION in_row_index, JSAMPIMAGE output_buf, JDIMENSION out_row_group_index)`
- **Purpose:** Top-level downsample dispatcher; iterates over all components and calls each component's selected method pointer.
- **Inputs:** Compression context, input/output image buffers and row indices.
- **Outputs/Return:** Fills `output_buf` for the current row group.
- **Side effects:** Delegates to per-component functions which may modify input rows (edge padding).
- **Calls:** `(*downsample->methods[ci])` for each component.
- **Notes:** Installed as `downsample->pub.downsample` by `jinit_downsampler`.

---

### int_downsample
- **Signature:** `METHODDEF void int_downsample(j_compress_ptr cinfo, jpeg_component_info *compptr, JSAMPARRAY input_data, JSAMPARRAY output_data)`
- **Purpose:** General-purpose integer-ratio box-filter downsample; averages `h_expand × v_expand` source pixels per output sample.
- **Inputs:** Compression context, component descriptor, input/output row arrays.
- **Outputs/Return:** Writes averaged samples to `output_data`.
- **Side effects:** Calls `expand_right_edge` on `input_data`.
- **Calls:** `expand_right_edge`, `GETJSAMPLE`.
- **Notes:** Not used for the common 1:1 or 2:1 cases; those have dedicated optimized routines.

---

### fullsize_downsample
- **Signature:** `METHODDEF void fullsize_downsample(...)`
- **Purpose:** 1:1 passthrough — copies rows from input to output and pads the right edge.
- **Calls:** `jcopy_sample_rows`, `expand_right_edge`.

---

### h2v1_downsample
- **Signature:** `METHODDEF void h2v1_downsample(...)`
- **Purpose:** 2:1 horizontal, 1:1 vertical; averages adjacent horizontal pixel pairs with alternating bias (0/1) to avoid systematic rounding error.
- **Calls:** `expand_right_edge`, `GETJSAMPLE`.

---

### h2v2_downsample
- **Signature:** `METHODDEF void h2v2_downsample(...)`
- **Purpose:** Standard 2:1 both axes; averages 2×2 pixel blocks with alternating bias (1/2).
- **Calls:** `expand_right_edge`, `GETJSAMPLE`.

---

### h2v2_smooth_downsample *(INPUT_SMOOTHING_SUPPORTED)*
- **Signature:** `METHODDEF void h2v2_smooth_downsample(...)`
- **Purpose:** Like `h2v2_downsample` but applies a 3×3 weighted neighbor smoothing filter (scaled integer arithmetic, factor `smoothing_factor/1024`) using one row of context above and below.
- **Side effects:** Requires `need_context_rows = TRUE`.
- **Notes:** Uses fixed-point scale factors (`memberscale`, `neighscale`); handles first and last column as special cases.

---

### fullsize_smooth_downsample *(INPUT_SMOOTHING_SUPPORTED)*
- **Signature:** `METHODDEF void fullsize_smooth_downsample(...)`
- **Purpose:** 1:1 passthrough with 8-neighbor weighted smoothing using context rows.
- **Notes:** `memberscale = 65536 - smoothing_factor * 512`, `neighscale = smoothing_factor * 64`.

---

### jinit_downsampler
- **Signature:** `GLOBAL void jinit_downsampler(j_compress_ptr cinfo)`
- **Purpose:** Module initializer; allocates `my_downsampler`, sets public method pointers, then selects the appropriate per-component downsample routine based on sampling factors.
- **Inputs:** Compression context (sampling factors, smoothing_factor already set).
- **Outputs/Return:** Installs `cinfo->downsample`.
- **Side effects:** Allocates from `JPOOL_IMAGE`; calls `ERREXIT` for unsupported configurations (CCIR601, fractional ratios).
- **Calls:** `alloc_small`, `ERREXIT`, `TRACEMS`.
- **Notes:** CCIR601 cosited sampling is explicitly rejected. Smoothing is silently disabled (with a trace message) for h2v1 and arbitrary-ratio cases.

## Control Flow Notes
Called once during compression startup by the prep-controller initialization chain. `jinit_downsampler` runs at init time; `sep_downsample` (via `pub.downsample`) is invoked once per row group during the main compression data pass. `start_pass_downsample` is a no-op placeholder.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMCOPY`, etc.)
- `jpeglib.h` / `jpegint.h` — `j_compress_ptr`, `jpeg_component_info`, `jpeg_downsampler`, `JSAMPARRAY`, `JDIMENSION`, `INT32`, `GETJSAMPLE`, `JMETHOD`, `DCTSIZE`, `MAX_COMPONENTS`
- `jcopy_sample_rows` — defined elsewhere (jutils.c); bulk row copy
- `ERREXIT`, `TRACEMS` — error/trace macros expanding to `cinfo->err` method calls
