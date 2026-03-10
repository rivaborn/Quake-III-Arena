# code/jpeg-6/jdapimin.c

## File Purpose
Implements the minimum public API for the JPEG decompression half of the IJG JPEG library. Provides object lifecycle management (create/destroy/abort), header reading, incremental input consumption, and decompression finalization routines.

## Core Responsibilities
- Initialize and zero-out a `jpeg_decompress_struct`, wiring up memory manager and input controller
- Destroy or abort a decompression object, releasing allocated resources
- Read and parse the JPEG header up to the first SOS marker via `jpeg_read_header`
- Drive the input state machine through `jpeg_consume_input`, handling DSTATE transitions
- Set default decompression parameters (colorspace, scaling, dithering, quantization)
- Install custom COM/APPn marker handler callbacks
- Finalize decompression (drain remaining input, release memory)

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `j_decompress_ptr` | typedef (pointer to struct) | Handle to a `jpeg_decompress_struct`; passed to all decompressor API calls |
| `jpeg_marker_parser_method` | typedef (function pointer) | Signature for application-supplied COM/APPn marker handlers |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_create_decompress
- **Signature:** `GLOBAL void jpeg_create_decompress(j_decompress_ptr cinfo)`
- **Purpose:** Initializes a new decompression instance: zeros the struct, creates the memory manager, initializes the marker reader and input controller, sets `global_state = DSTATE_START`.
- **Inputs:** `cinfo` — caller-allocated decompress struct with `err` already set.
- **Outputs/Return:** void; `cinfo` is fully initialized.
- **Side effects:** Allocates internal memory manager state; calls `jinit_marker_reader`, `jinit_input_controller`.
- **Calls:** `jinit_memory_mgr`, `jinit_marker_reader`, `jinit_input_controller`
- **Notes:** Error manager must be set before call; it is preserved across the `MEMZERO`.

### jpeg_destroy_decompress
- **Signature:** `GLOBAL void jpeg_destroy_decompress(j_decompress_ptr cinfo)`
- **Purpose:** Fully destroys the decompression object and frees all memory.
- **Inputs:** `cinfo`
- **Outputs/Return:** void
- **Side effects:** All memory pools freed; object unusable after call.
- **Calls:** `jpeg_destroy` (common routine in `jcomapi.c`)

### jpeg_abort_decompress
- **Signature:** `GLOBAL void jpeg_abort_decompress(j_decompress_ptr cinfo)`
- **Purpose:** Aborts an in-progress decompression without destroying the object; resets to `DSTATE_START`.
- **Calls:** `jpeg_abort` (common routine)

### jpeg_set_marker_processor
- **Signature:** `GLOBAL void jpeg_set_marker_processor(j_decompress_ptr cinfo, int marker_code, jpeg_marker_parser_method routine)`
- **Purpose:** Installs a custom callback for a COM or APPn (APP0–APP15) JPEG marker.
- **Inputs:** `marker_code` — `JPEG_COM` or `JPEG_APP0`..`JPEG_APP0+15`; `routine` — function pointer.
- **Outputs/Return:** void; writes into `cinfo->marker->process_COM` or `process_APPn[]`.
- **Side effects:** Modifies marker reader vtable.
- **Notes:** Fires `ERREXIT1` for unrecognized marker codes.

### default_decompress_parms *(LOCAL)*
- **Signature:** `LOCAL void default_decompress_parms(j_decompress_ptr cinfo)`
- **Purpose:** Infers JPEG colorspace from component count and JFIF/Adobe markers; sets output colorspace, scaling (1:1), DCT method, dithering, quantization defaults.
- **Inputs:** `cinfo` with header already parsed.
- **Outputs/Return:** void; writes multiple fields in `cinfo`.
- **Notes:** Heuristic colorspace detection; warns on unknown Adobe transform codes; `QUANT_2PASS_SUPPORTED` controls `two_pass_quantize` default.

### jpeg_read_header
- **Signature:** `GLOBAL int jpeg_read_header(j_decompress_ptr cinfo, boolean require_image)`
- **Purpose:** Reads the JPEG datastream up to the first SOS marker; saves tables and parameters.
- **Inputs:** `require_image` — if TRUE, treats a tables-only stream as an error.
- **Outputs/Return:** `JPEG_HEADER_OK`, `JPEG_HEADER_TABLES_ONLY`, or `JPEG_SUSPENDED`.
- **Side effects:** May call `jpeg_abort` on tables-only path; advances `global_state` to `DSTATE_READY` (via `jpeg_consume_input`).
- **Calls:** `jpeg_consume_input`, `jpeg_abort`

### jpeg_consume_input
- **Signature:** `GLOBAL int jpeg_consume_input(j_decompress_ptr cinfo)`
- **Purpose:** State machine driving JPEG input: initializes source on `DSTATE_START`, dispatches to `inputctl->consume_input` for header/scan states, transitions to `DSTATE_READY` upon finding SOS.
- **Outputs/Return:** One of `JPEG_SUSPENDED`, `JPEG_REACHED_SOS`, `JPEG_REACHED_EOI`, `JPEG_ROW_COMPLETED`, `JPEG_SCAN_COMPLETED`.
- **Side effects:** Calls `src->init_source` once; calls `inputctl->reset_input_controller`; calls `default_decompress_parms` on SOS.
- **Calls:** `cinfo->inputctl->reset_input_controller`, `cinfo->src->init_source`, `cinfo->inputctl->consume_input`, `default_decompress_parms`

### jpeg_finish_decompress
- **Signature:** `GLOBAL boolean jpeg_finish_decompress(j_decompress_ptr cinfo)`
- **Purpose:** Finalizes decompression: terminates output pass, drains remaining input to EOI, calls `src->term_source`, then resets via `jpeg_abort`.
- **Outputs/Return:** `FALSE` if suspended (resumable); `TRUE` on completion.
- **Side effects:** Calls `master->finish_output_pass`, `src->term_source`, `jpeg_abort`; resets object to `DSTATE_START`.
- **Notes:** Safe to call again after suspension; errors on incorrect state or insufficient scanlines output.

## Control Flow Notes
This file covers the **init → header-read → finish** lifecycle of a decompressor. `jpeg_create_decompress` is called once at startup. `jpeg_read_header` / `jpeg_consume_input` are called during the header-parsing phase (before frame decode). `jpeg_finish_decompress` is called at shutdown. The file does not participate in the per-frame scanline decode loop (that is `jdapistd.c`).

## External Dependencies
- **`jinclude.h`** — platform portability macros (`MEMZERO`, `SIZEOF`)
- **`jpeglib.h`** — all public JPEG types and struct definitions
- **`jpegint.h`** (via `JPEG_INTERNALS`) — internal module interfaces
- **`jerror.h`** (via `JPEG_INTERNALS`) — `ERREXIT1`, `WARNMS1`, `TRACEMS3` macros
- **Defined elsewhere:** `jinit_memory_mgr`, `jinit_marker_reader`, `jinit_input_controller` (module init functions); `jpeg_destroy`, `jpeg_abort` (`jcomapi.c`); `inputctl->consume_input`, `src->init_source`, `src->term_source`, `master->finish_output_pass` (subobject vtable methods)
