# code/jpeg-6/jdpostct.c

## File Purpose
Implements the JPEG decompression postprocessing controller, which manages the pipeline stage between upsampling/color-conversion and color quantization/reduction. It buffers decoded pixel data in either a single strip or a full-image virtual array depending on the quantization pass mode.

## Core Responsibilities
- Initialize and own the strip buffer or full-image virtual array used between upsample and quantize stages
- Select the correct processing function pointer (`post_process_data`) based on the current pass mode
- Drive the upsample→quantize pipeline for one-pass color quantization
- Buffer full-image rows during the first pass of two-pass color quantization (prepass, no output emitted)
- Re-read buffered rows and quantize+emit them during the second pass of two-pass quantization
- Short-circuit the postprocessing stage entirely when no color quantization is needed (delegate directly to upsampler)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_post_controller` | struct | Private extension of `jpeg_d_post_controller`; holds the strip/virtual buffer, strip height, and two-pass row tracking fields |
| `my_post_ptr` | typedef | Convenience pointer to `my_post_controller` |

## Global / File-Static State
None.

## Key Functions / Methods

### `start_pass_dpost`
- **Signature:** `METHODDEF void start_pass_dpost(j_decompress_ptr cinfo, J_BUF_MODE pass_mode)`
- **Purpose:** Called at the start of each decompression pass; selects the correct `post_process_data` function pointer and resets row counters.
- **Inputs:** `cinfo` — decompressor state; `pass_mode` — one of `JBUF_PASS_THRU`, `JBUF_SAVE_AND_PASS`, `JBUF_CRANK_DEST`
- **Outputs/Return:** void; sets `post->pub.post_process_data` function pointer
- **Side effects:** May allocate/access the virtual sarray buffer (`access_virt_sarray`) if `buffer` is NULL in pass-through+quantize mode; resets `starting_row` and `next_row` to 0
- **Calls:** `cinfo->mem->access_virt_sarray`, `ERREXIT`
- **Notes:** If quantization is disabled, `post_process_data` is set directly to `cinfo->upsample->upsample`, bypassing this module's functions entirely.

### `post_process_1pass`
- **Signature:** `METHODDEF void post_process_1pass(j_decompress_ptr cinfo, JSAMPIMAGE input_buf, JDIMENSION *in_row_group_ctr, JDIMENSION in_row_groups_avail, JSAMPARRAY output_buf, JDIMENSION *out_row_ctr, JDIMENSION out_rows_avail)`
- **Purpose:** Processes one chunk of rows for single-pass color quantization; upsample into the strip buffer then immediately quantize to output.
- **Inputs:** Compressed component input rows, available input row group count, output row buffer and counters
- **Outputs/Return:** void; advances `*out_row_ctr` by rows emitted
- **Side effects:** Writes into `post->buffer`; calls upsampler and color quantizer
- **Calls:** `cinfo->upsample->upsample`, `cinfo->cquantize->color_quantize`
- **Notes:** Output is clamped to `min(out_rows_avail - *out_row_ctr, strip_height)` to avoid overrun.

### `post_process_prepass` *(QUANT_2PASS_SUPPORTED)*
- **Signature:** `METHODDEF void post_process_prepass(j_decompress_ptr cinfo, ...)`
- **Purpose:** First pass of two-pass quantization; upsample into the full-image virtual buffer and let the quantizer scan the data to build its color histogram. No pixel data is emitted to the output buffer.
- **Inputs/Outputs:** Same signature as above; `output_buf` is unused (NULL passed to quantizer)
- **Side effects:** Writes into `post->whole_image` via `access_virt_sarray`; advances `starting_row`/`next_row`; advances `*out_row_ctr` so the outer loop knows progress
- **Calls:** `cinfo->mem->access_virt_sarray`, `cinfo->upsample->upsample`, `cinfo->cquantize->color_quantize`
- **Notes:** Repositions the virtual buffer window at strip boundaries (`next_row == 0`).

### `post_process_2pass` *(QUANT_2PASS_SUPPORTED)*
- **Signature:** `METHODDEF void post_process_2pass(j_decompress_ptr cinfo, ...)`
- **Purpose:** Second pass of two-pass quantization; reads back rows from the full-image virtual buffer (read-only) and emits quantized pixels to the output scanline buffer.
- **Inputs/Outputs:** `input_buf` / `in_row_group_ctr` are unused; reads from `post->whole_image`; fills `output_buf`
- **Side effects:** Reads via `access_virt_sarray` (writable=FALSE); calls quantizer; advances `starting_row`/`next_row`/`*out_row_ctr`
- **Calls:** `cinfo->mem->access_virt_sarray`, `cinfo->cquantize->color_quantize`
- **Notes:** Must manually check for image bottom (`cinfo->output_height`) because the upsampler is not invoked in this pass.

### `jinit_d_post_controller`
- **Signature:** `GLOBAL void jinit_d_post_controller(j_decompress_ptr cinfo, boolean need_full_buffer)`
- **Purpose:** Allocates and initializes the postprocessing controller object; creates the strip buffer or full-image virtual array as needed.
- **Inputs:** `cinfo` — decompressor; `need_full_buffer` — TRUE to allocate full-image storage for two-pass quantization
- **Outputs/Return:** void; sets `cinfo->post`
- **Side effects:** Allocates `my_post_controller` from JPOOL_IMAGE; conditionally allocates sarray or virtual sarray via memory manager
- **Calls:** `cinfo->mem->alloc_small`, `cinfo->mem->request_virt_sarray`, `cinfo->mem->alloc_sarray`, `ERREXIT`, `jround_up`
- **Notes:** Strip height is set to `max_v_samp_factor`, which aligns with the upsampler's natural output granularity.

## Control Flow Notes
`jinit_d_post_controller` is called during decompression initialization. `start_pass_dpost` is called at the start of each output pass. The selected `post_process_data` function is then invoked repeatedly by the main controller per strip/row group during the decompression output loop. For no-quantize paths, this controller adds zero overhead after `start_pass_dpost` by delegating directly to the upsampler.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `jpeg_decompress_struct`, `jpeg_d_post_controller`, `jvirt_sarray_ptr`, `JSAMPIMAGE`, `JSAMPARRAY`, `JDIMENSION`, `J_BUF_MODE`, `JPOOL_IMAGE`
- **Defined elsewhere:** `jround_up` (math utility), `ERREXIT` (error macro), `cinfo->upsample->upsample`, `cinfo->cquantize->color_quantize`, `cinfo->mem->*` (memory manager vtable)
