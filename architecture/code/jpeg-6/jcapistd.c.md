# code/jpeg-6/jcapistd.c

## File Purpose
Implements the standard JPEG compression API entry points for full-compression workflows: initializing a compression session, writing scanlines of image data, and writing raw downsampled data. Intentionally separated from `jcapimin.c` to prevent linking the full compressor into transcoding-only applications.

## Core Responsibilities
- Initialize a compression session and activate all encoder submodules
- Accept and process scanline-format image data from the caller
- Accept and process pre-downsampled (raw) image data in iMCU-row units
- Track and report scanline progress via an optional progress monitor hook
- Enforce call-sequence validity via `global_state` checks

## Key Types / Data Structures
None defined in this file; all types come from `jpeglib.h`.

| Name | Kind | Purpose |
|---|---|---|
| `j_compress_ptr` | typedef (pointer to struct) | Handle to the compression instance (`jpeg_compress_struct *`) |
| `JSAMPARRAY` | typedef | 2-D array of pixel samples; input to `jpeg_write_scanlines` |
| `JSAMPIMAGE` | typedef | 3-D sample array (per color channel); input to `jpeg_write_raw_data` |
| `JDIMENSION` | typedef | Unsigned integer for image/scanline dimensions |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_start_compress
- **Signature:** `GLOBAL void jpeg_start_compress(j_compress_ptr cinfo, boolean write_all_tables)`
- **Purpose:** Initializes a compression run. Resets error and destination modules, selects and initializes all active encoder submodules, and prepares for the first encoding pass.
- **Inputs:** `cinfo` — compression instance; `write_all_tables` — if TRUE, marks all Huffman/quantization tables to be written (prevents abbreviated-stream mistakes).
- **Outputs/Return:** `void`
- **Side effects:** Sets `cinfo->next_scanline = 0`; sets `cinfo->global_state` to `CSTATE_SCANNING` or `CSTATE_RAW_OK`; calls `jpeg_suppress_tables`, `reset_error_mgr`, `init_destination`, `jinit_compress_master`, `prepare_for_pass`.
- **Calls:** `jpeg_suppress_tables`, `cinfo->err->reset_error_mgr`, `cinfo->dest->init_destination`, `jinit_compress_master`, `cinfo->master->prepare_for_pass`
- **Notes:** Errors if `global_state != CSTATE_START`. The `write_all_tables=TRUE` convention is the recommended safe default.

### jpeg_write_scanlines
- **Signature:** `GLOBAL JDIMENSION jpeg_write_scanlines(j_compress_ptr cinfo, JSAMPARRAY scanlines, JDIMENSION num_lines)`
- **Purpose:** Feeds one or more rows of interleaved (non-downsampled) pixel data into the compressor.
- **Inputs:** `cinfo` — compression instance; `scanlines` — array of pointers to row buffers; `num_lines` — number of rows supplied.
- **Outputs/Return:** Number of scanlines actually consumed (`JDIMENSION`); may be less than `num_lines` if the destination module suspends.
- **Side effects:** Advances `cinfo->next_scanline`; updates `cinfo->progress` counters; may trigger `pass_startup` on the first call; silently clamps `num_lines` to remaining rows.
- **Calls:** `cinfo->progress->progress_monitor` (if set), `cinfo->master->pass_startup` (if `call_pass_startup`), `cinfo->main->process_data`
- **Notes:** Warns (`JWRN_TOO_MUCH_DATA`) if called after all rows have been written; extra scanlines in the last valid call are silently ignored.

### jpeg_write_raw_data
- **Signature:** `GLOBAL JDIMENSION jpeg_write_raw_data(j_compress_ptr cinfo, JSAMPIMAGE data, JDIMENSION num_lines)`
- **Purpose:** Alternate entry point for callers supplying pre-downsampled data; processes exactly one iMCU row per call.
- **Inputs:** `cinfo` — compression instance; `data` — per-component 2-D sample arrays; `num_lines` — must be ≥ `max_v_samp_factor * DCTSIZE`.
- **Outputs/Return:** Lines consumed (= `lines_per_iMCU_row`) on success; `0` if the compressor suspends or input is exhausted.
- **Side effects:** Advances `cinfo->next_scanline` by `lines_per_iMCU_row`; updates progress counters; may trigger `pass_startup`.
- **Calls:** `cinfo->progress->progress_monitor` (if set), `cinfo->master->pass_startup` (if `call_pass_startup`), `cinfo->coef->compress_data`
- **Notes:** Errors if `global_state != CSTATE_RAW_OK` or if `num_lines < lines_per_iMCU_row`. Only callable when `cinfo->raw_data_in == TRUE`.

## Control Flow Notes
These functions represent the **data-feeding phase** of the compression pipeline:
1. **Init:** `jpeg_start_compress` is called once before any data is fed; it bootstraps all submodule objects.
2. **Loop:** The application repeatedly calls `jpeg_write_scanlines` (or `jpeg_write_raw_data`) until `cinfo->next_scanline >= cinfo->image_height`.
3. **Finish:** `jpeg_finish_compress` (defined in `jcapimin.c`, not here) is called after all scanlines are written to flush and finalize the bitstream.

## External Dependencies
- `jinclude.h` — system header portability layer (`MEMZERO`, `MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` — all public JPEG types and the `jpeg_compress_struct` definition
- `jpegint.h` (via `JPEG_INTERNALS`) — internal submodule interface structs (`jpeg_comp_master`, `jpeg_c_main_controller`, `jpeg_c_coef_controller`, etc.)
- `jerror.h` (via `JPEG_INTERNALS`) — `ERREXIT1`, `WARNMS`, `ERREXIT` macros and error codes
- **Defined elsewhere:** `jinit_compress_master` (jcmaster.c), `jpeg_suppress_tables` (jcparam.c), all vtable method implementations (`process_data`, `compress_data`, `pass_startup`, `prepare_for_pass`, `progress_monitor`, `init_destination`, `reset_error_mgr`)
