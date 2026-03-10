# code/jpeg-6/jcmainct.c

## File Purpose
Implements the main buffer controller for the JPEG compression pipeline. It sits between the pre-processor (downsampling/color conversion) and the DCT/entropy coefficient compressor, managing the intermediate strip buffer of downsampled JPEG-colorspace data.

## Core Responsibilities
- Allocate and manage per-component strip buffers (or optional full-image virtual arrays) to hold downsampled data
- Initialize pass state (iMCU row counters, buffer mode) at the start of each compression pass
- Drive the data flow loop: pull rows from the preprocessor into the strip buffer, then push complete iMCU rows to the coefficient compressor
- Handle compressor suspension (output-not-consumed) by backing up the input row counter and retrying on the next call
- Expose the `start_pass` and `process_data` method pointers on `jpeg_c_main_controller`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `my_main_controller` | struct | Private extension of `jpeg_c_main_controller`; holds iMCU row/row-group counters, suspension flag, pass mode, and per-component buffer pointers |
| `my_main_ptr` | typedef | Pointer alias for `my_main_controller*`; used for downcasting `cinfo->main` |

## Global / File-Static State
None.

## Key Functions / Methods

### `start_pass_main`
- **Signature:** `METHODDEF void start_pass_main(j_compress_ptr cinfo, J_BUF_MODE pass_mode)`
- **Purpose:** Resets counters and assigns the correct `process_data` function pointer for the requested pass mode.
- **Inputs:** `cinfo` — compress object; `pass_mode` — `JBUF_PASS_THRU` (or full-buffer modes if compiled in)
- **Outputs/Return:** void
- **Side effects:** Writes `cur_iMCU_row`, `rowgroup_ctr`, `suspended`, `pass_mode`, `pub.process_data` into `jmain`
- **Calls:** `ERREXIT` on invalid mode
- **Notes:** No-ops entirely when `cinfo->raw_data_in` is set (raw-data path bypasses this module).

### `process_data_simple_main`
- **Signature:** `METHODDEF void process_data_simple_main(j_compress_ptr cinfo, JSAMPARRAY input_buf, JDIMENSION *in_row_ctr, JDIMENSION in_rows_avail)`
- **Purpose:** Normal pass-through path. Fills the strip buffer one DCT-row-group at a time via the preprocessor, then submits full iMCU rows to the coefficient compressor.
- **Inputs:** `input_buf` — raw scanlines from application; `in_row_ctr`/`in_rows_avail` — tracks how many input rows have been consumed
- **Outputs/Return:** void; advances `*in_row_ctr` and internal counters
- **Side effects:** Modifies `jmain->rowgroup_ctr`, `jmain->cur_iMCU_row`, `jmain->suspended`, and `*in_row_ctr`
- **Calls:** `cinfo->prep->pre_process_data`, `cinfo->coef->compress_data`
- **Notes:** Suspension hack: if `compress_data` returns false (output buffer full), decrements `*in_row_ctr` to re-present the last row on re-entry; reverses the decrement once the compressor succeeds.

### `process_data_buffer_main` *(compiled only if `FULL_MAIN_BUFFER_SUPPORTED`)*
- **Signature:** `METHODDEF void process_data_buffer_main(j_compress_ptr cinfo, JSAMPARRAY input_buf, JDIMENSION *in_row_ctr, JDIMENSION in_rows_avail)`
- **Purpose:** Full-image-buffer variant; realigns virtual sample arrays per iMCU row and supports `JBUF_SAVE_SOURCE` / `JBUF_CRANK_DEST` / `JBUF_SAVE_AND_PASS` multi-pass modes.
- **Inputs/Outputs:** Same pattern as simple variant; additionally accesses `main->whole_image[]` virtual arrays
- **Side effects:** Calls `cinfo->mem->access_virt_sarray` to page virtual buffers; same suspension handling as simple path
- **Calls:** `cinfo->mem->access_virt_sarray`, `cinfo->prep->pre_process_data`, `cinfo->coef->compress_data`

### `jinit_c_main_controller`
- **Signature:** `GLOBAL void jinit_c_main_controller(j_compress_ptr cinfo, boolean need_full_buffer)`
- **Purpose:** Allocates the `my_main_controller` struct, installs it as `cinfo->main`, and allocates per-component strip buffers (or full virtual arrays if requested).
- **Inputs:** `cinfo`; `need_full_buffer` — TRUE requests full-image virtual arrays (requires `FULL_MAIN_BUFFER_SUPPORTED`)
- **Outputs/Return:** void; populates `cinfo->main`
- **Side effects:** Allocates from `JPOOL_IMAGE`; calls `cinfo->mem->alloc_small`, `alloc_sarray`, or `request_virt_sarray`
- **Notes:** Returns early without allocating buffers when `cinfo->raw_data_in` is TRUE.

## Control Flow Notes
Called once during compression init via `jinit_c_main_controller`. Each compression pass begins with `start_pass_main` (invoked by the master controller). During the compression loop, `process_data` is called repeatedly by `jpeg_write_scanlines` until all iMCU rows are consumed. The module is entirely bypassed when `raw_data_in` is set.

## External Dependencies
- `jinclude.h` — system includes, `MEMZERO`/`MEMCOPY`, `SIZEOF`
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `jpeg_c_main_controller`, `jpeg_component_info`, `JDIMENSION`, `JSAMPARRAY`, `J_BUF_MODE`, `DCTSIZE`, `MAX_COMPONENTS`, `jround_up`
- `ERREXIT` — error macro defined in `jerror.h`
- `cinfo->prep->pre_process_data` — defined in `jcprepct.c`
- `cinfo->coef->compress_data` — defined in `jccoefct.c`
- `cinfo->mem->alloc_small`, `alloc_sarray`, `request_virt_sarray`, `access_virt_sarray` — defined in `jmemmgr.c`
