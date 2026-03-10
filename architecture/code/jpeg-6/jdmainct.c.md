# code/jpeg-6/jdmainct.c

## File Purpose
Implements the main buffer controller for the JPEG decompressor, sitting between the coefficient decoder and the post-processor. It manages downsampled sample data in JPEG colorspace, optionally providing context rows (above/below neighbors) required by fancy upsampling algorithms.

## Core Responsibilities
- Allocate and manage the intermediate sample buffer between coefficient decode and post-processing
- Deliver iMCU row data to the post-processor as row groups
- Optionally maintain a "funny pointer" scheme to provide context rows without copying data
- Handle image top/bottom boundary conditions by duplicating edge sample rows
- Support a two-pass quantization crank mode that bypasses the main buffer entirely
- Initialize the `jpeg_d_main_controller` sub-object and wire it into `cinfo->main`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_main_controller` | struct | Private extension of `jpeg_d_main_controller`; holds workspace buffers, state machine variables, and the two "funny" pointer arrays |
| `my_main_ptr` | typedef | Convenience pointer to `my_main_controller` |

## Global / File-Static State
None.

## Key Functions / Methods

### jinit_d_main_controller
- **Signature:** `GLOBAL void jinit_d_main_controller(j_decompress_ptr cinfo, boolean need_full_buffer)`
- **Purpose:** Allocates and initializes the main buffer controller; allocates per-component sample workspace.
- **Inputs:** `cinfo` — decompressor state; `need_full_buffer` — must be FALSE (full-image buffering unsupported here).
- **Outputs/Return:** void; sets `cinfo->main`.
- **Side effects:** Allocates `my_main_controller` and per-component `JSAMPARRAY` buffers via `cinfo->mem->alloc_small` / `alloc_sarray` in `JPOOL_IMAGE`. Calls `alloc_funny_pointers` when context rows are needed.
- **Calls:** `alloc_funny_pointers`, `ERREXIT`, `cinfo->mem->alloc_small`, `cinfo->mem->alloc_sarray`.
- **Notes:** Errors if `need_full_buffer` is TRUE or if `need_context_rows` is TRUE with `min_DCT_scaled_size < 2`.

### start_pass_main
- **Signature:** `METHODDEF void start_pass_main(j_decompress_ptr cinfo, J_BUF_MODE pass_mode)`
- **Purpose:** Initializes controller state at the start of each decompression pass; selects the appropriate `process_data` function pointer.
- **Inputs:** `pass_mode` — one of `JBUF_PASS_THRU` or `JBUF_CRANK_DEST`.
- **Outputs/Return:** void.
- **Side effects:** Sets `jmain->pub.process_data`, resets `buffer_full`, `rowgroup_ctr`; calls `make_funny_pointers` for context mode.
- **Calls:** `make_funny_pointers`, `ERREXIT`.

### process_data_simple_main
- **Signature:** `METHODDEF void process_data_simple_main(j_decompress_ptr cinfo, JSAMPARRAY output_buf, JDIMENSION *out_row_ctr, JDIMENSION out_rows_avail)`
- **Purpose:** Processes one iMCU row at a time with no context rows; used when `need_context_rows` is FALSE.
- **Inputs:** Output pixel buffer and row counters.
- **Outputs/Return:** void; advances `*out_row_ctr`.
- **Side effects:** Calls coefficient decoder to fill buffer; calls post-processor; resets `buffer_full` when consumed.
- **Calls:** `cinfo->coef->decompress_data`, `cinfo->post->post_process_data`.
- **Notes:** May suspend (return early) if decoder suspends.

### process_data_context_main
- **Signature:** `METHODDEF void process_data_context_main(j_decompress_ptr cinfo, JSAMPARRAY output_buf, JDIMENSION *out_row_ctr, JDIMENSION out_rows_avail)`
- **Purpose:** Processes iMCU rows with above/below context rows provided via the funny-pointer scheme; state machine across three states (`CTX_PREPARE_FOR_IMCU`, `CTX_PROCESS_IMCU`, `CTX_POSTPONED_ROW`).
- **Side effects:** Advances `iMCU_row_ctr`; calls `set_wraparound_pointers` after first iMCU row; calls `set_bottom_pointers` at image bottom; toggles `whichptr`.
- **Calls:** `cinfo->coef->decompress_data`, `cinfo->post->post_process_data`, `set_bottom_pointers`, `set_wraparound_pointers`.

### alloc_funny_pointers / make_funny_pointers / set_wraparound_pointers / set_bottom_pointers
- `alloc_funny_pointers`: allocates the two redundant `xbuffer` pointer arrays (M+4 row groups each, offset by one rgroup for negative indexing).
- `make_funny_pointers`: fills the pointer arrays in the interleaved order described in the file's header comment.
- `set_wraparound_pointers`: transitions pointer lists from top-of-image state to normal wraparound state after first iMCU row.
- `set_bottom_pointers`: duplicates the last real sample row into padding slots and computes `rowgroups_avail` for the final iMCU row.

## Control Flow Notes
- Called during **init**: `jinit_d_main_controller` is invoked during decompressor initialization.
- Called at **pass start**: `start_pass_main` (via `pub.start_pass`) selects the active `process_data` variant.
- Called **per output batch**: the chosen `process_data` function is called repeatedly by the decompressor master loop until all scanlines are produced. The simple path is a single-level call chain (coef → main → post); the context path is a three-state machine that can suspend and resume at each call.
- Raw-data output mode bypasses this controller entirely.

## External Dependencies
- **Includes:** `jinclude.h` (platform portability, `SIZEOF`, `MEMCOPY`), `jpeglib.h` (all JPEG types and sub-object interfaces, pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`)
- **Defined elsewhere:**
  - `jpeg_d_main_controller` (declared in `jpegint.h`)
  - `cinfo->coef->decompress_data` — coefficient controller
  - `cinfo->post->post_process_data` — post-processing controller
  - `cinfo->upsample->need_context_rows` — upsampler flag
  - `cinfo->mem->alloc_small`, `alloc_sarray` — memory manager
  - `ERREXIT`, `JPOOL_IMAGE`, `JBUF_PASS_THRU`, `JBUF_CRANK_DEST`, `METHODDEF`, `LOCAL`, `GLOBAL`, `JPP` — macros from `jpegint.h`/`jmorecfg.h`
