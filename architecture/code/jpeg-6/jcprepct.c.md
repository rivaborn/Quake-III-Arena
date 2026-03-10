# code/jpeg-6/jcprepct.c

## File Purpose
Implements the JPEG compression preprocessing controller, which manages the pipeline stage between raw input scanlines and the downsampler. It orchestrates color conversion, intermediate buffering, and vertical edge padding to satisfy the downsampler's row-group alignment requirements.

## Core Responsibilities
- Initialize and own the `my_prep_controller` object attached to `cinfo->prep`
- Accept raw input scanlines and drive the color converter (`cinfo->cconvert->color_convert`)
- Buffer color-converted rows until a full row group is ready for downsampling
- Invoke the downsampler (`cinfo->downsample->downsample`) on complete row groups
- Pad the bottom edge of the image by replicating the last real pixel row
- Pad downsampler output to a full iMCU height at image bottom
- Optionally support context-row mode (for input smoothing), providing wraparound row-pointer buffers

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `my_prep_controller` | struct | Private preprocessing controller; extends `jpeg_c_prep_controller` with color conversion buffer, row counters, and context-mode state |
| `my_prep_ptr` | typedef | Convenience pointer to `my_prep_controller` |

## Global / File-Static State
None.

## Key Functions / Methods

### `start_pass_prep`
- **Signature:** `METHODDEF void start_pass_prep(j_compress_ptr cinfo, J_BUF_MODE pass_mode)`
- **Purpose:** Resets preprocessing state at the start of each compression pass.
- **Inputs:** `cinfo` — compress object; `pass_mode` — must be `JBUF_PASS_THRU`
- **Outputs/Return:** void
- **Side effects:** Writes `prep->rows_to_go`, `prep->next_buf_row`, and (conditionally) `prep->this_row_group`, `prep->next_buf_stop`
- **Calls:** `ERREXIT`
- **Notes:** Errors out on any mode other than pass-through; no multi-pass buffering supported.

### `expand_bottom_edge`
- **Signature:** `LOCAL void expand_bottom_edge(JSAMPARRAY image_data, JDIMENSION num_cols, int input_rows, int output_rows)`
- **Purpose:** Pads a sample array vertically by replicating row `input_rows-1` into rows `input_rows..output_rows-1`.
- **Inputs:** Array of row pointers, column count, first row to fill, last row (exclusive)
- **Outputs/Return:** void (modifies `image_data` in-place)
- **Side effects:** Calls `jcopy_sample_rows` for each padding row
- **Calls:** `jcopy_sample_rows` (defined elsewhere)
- **Notes:** Used both on the color buffer (input to downsampler) and on the downsampler output buffer.

### `pre_process_data`
- **Signature:** `METHODDEF void pre_process_data(j_compress_ptr cinfo, JSAMPARRAY input_buf, JDIMENSION *in_row_ctr, JDIMENSION in_rows_avail, JSAMPIMAGE output_buf, JDIMENSION *out_row_group_ctr, JDIMENSION out_row_groups_avail)`
- **Purpose:** Non-context processing path: color-converts input rows into `color_buf`, then downsamples when a full row group is accumulated.
- **Inputs:** Input scanline array + counters, output row-group array + counters
- **Outputs/Return:** void; advances `*in_row_ctr` and `*out_row_group_ctr`
- **Side effects:** Mutates `prep->next_buf_row`, `prep->rows_to_go`; calls color converter and downsampler; may pad bottom edge
- **Calls:** `cinfo->cconvert->color_convert`, `expand_bottom_edge`, `cinfo->downsample->downsample`
- **Notes:** Exits early once the entire image bottom is padded and output is filled to `out_row_groups_avail`.

### `pre_process_context`
- **Signature:** `METHODDEF void pre_process_context(j_compress_ptr cinfo, JSAMPARRAY input_buf, JDIMENSION *in_row_ctr, JDIMENSION in_rows_avail, JSAMPIMAGE output_buf, JDIMENSION *out_row_group_ctr, JDIMENSION out_row_groups_avail)`
- **Purpose:** Context-row processing path (compiled only when `CONTEXT_ROWS_SUPPORTED`): maintains a 3-row-group circular buffer with wraparound pointers so the downsampler can access one row group of context above and below.
- **Inputs/Outputs:** Same as `pre_process_data`
- **Side effects:** Manages `prep->this_row_group`, `prep->next_buf_row`, `prep->next_buf_stop` with modular arithmetic against `buf_height = max_v_samp_factor * 3`; pads top edge on first pass by copying row 0 into negative indices
- **Calls:** `cinfo->cconvert->color_convert`, `jcopy_sample_rows`, `expand_bottom_edge`, `cinfo->downsample->downsample`
- **Notes:** Top-of-image padding writes into `color_buf[ci][-row]`, relying on the fake pointer array set up by `create_context_buffer`.

### `create_context_buffer`
- **Signature:** `LOCAL void create_context_buffer(j_compress_ptr cinfo)`
- **Purpose:** Allocates the wraparound color-conversion buffer for context mode: 3 real row groups plus 2 extra virtual row groups (above/below) whose pointers wrap to the opposite end.
- **Inputs:** `cinfo`
- **Outputs/Return:** void; sets `prep->color_buf[ci]` for each component
- **Side effects:** Allocates from `JPOOL_IMAGE` via `cinfo->mem->alloc_small` and `cinfo->mem->alloc_sarray`
- **Calls:** `cinfo->mem->alloc_small`, `cinfo->mem->alloc_sarray`, `MEMCOPY`
- **Notes:** The 5-row-group fake pointer array enables negative-index access (context rows above image top).

### `jinit_c_prep_controller`
- **Signature:** `GLOBAL void jinit_c_prep_controller(j_compress_ptr cinfo, boolean need_full_buffer)`
- **Purpose:** Allocates and initializes the preprocessing controller; selects the simple or context processing path based on `cinfo->downsample->need_context_rows`.
- **Inputs:** `cinfo`; `need_full_buffer` must be FALSE
- **Outputs/Return:** void; sets `cinfo->prep`
- **Side effects:** Allocates `my_prep_controller` from `JPOOL_IMAGE`; allocates per-component `color_buf` arrays; installs method pointers
- **Calls:** `ERREXIT`, `cinfo->mem->alloc_small`, `create_context_buffer`, `cinfo->mem->alloc_sarray`
- **Notes:** Called once during compression startup from the master initialization sequence.

## Control Flow Notes
This module sits between the main controller (which feeds raw scanlines) and the downsampler. During each compression pass, `pre_process_data` (or `pre_process_context`) is called repeatedly by the main controller until the full image is consumed. It is purely a **compression-time preprocessing** module with no role in decompression, rendering, or frame updates.

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, system headers)
- `jpeglib.h` / `jpegint.h` — `jpeg_compress_struct`, `jpeg_component_info`, `jpeg_c_prep_controller`, `JSAMPARRAY`, `JSAMPIMAGE`, `JDIMENSION`, `JPOOL_IMAGE`, `DCTSIZE`
- `jcopy_sample_rows` — defined elsewhere (likely `jutils.c`); copies rows within a sample array
- `cinfo->cconvert->color_convert` — color space converter, defined elsewhere
- `cinfo->downsample->downsample` — downsampling module, defined elsewhere
- `ERREXIT`, `MIN` — macros from JPEG error/utility headers
