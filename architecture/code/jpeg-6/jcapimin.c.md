# code/jpeg-6/jcapimin.c

## File Purpose
Implements the minimum application interface for the JPEG compression half of the IJG JPEG library. Provides the core lifecycle functions (create, destroy, abort) and essential compression control functions (finish, write marker, write tables) needed for both normal compression and transcoding scenarios.

## Core Responsibilities
- Initialize and zero a `jpeg_compress_struct`, set up the memory manager, and transition to `CSTATE_START`
- Destroy or abort a compression object by delegating to common routines
- Mark quantization and Huffman tables as sent or unsent (suppress/un-suppress)
- Drive any remaining multi-pass compression work and finalize the JPEG bitstream (write EOI, flush destination)
- Write arbitrary JPEG markers (COM/APPn) between `jpeg_start_compress` and the first scanline
- Write an abbreviated table-only JPEG datastream without image data

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `jpeg_compress_struct` (`j_compress_ptr`) | struct | Master compression state object; holds all parameters, submodule pointers, and scan state |
| `JQUANT_TBL` | struct | DCT quantization table; `sent_table` flag controls re-emission |
| `JHUFF_TBL` | struct | Huffman coding table; `sent_table` flag controls re-emission |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_create_compress
- **Signature:** `GLOBAL void jpeg_create_compress(j_compress_ptr cinfo)`
- **Purpose:** Initializes a fresh compression instance; zeros the struct (preserving the error manager), sets `is_decompressor = FALSE`, initializes the memory manager, and sets `global_state = CSTATE_START`.
- **Inputs:** `cinfo` — caller-allocated `jpeg_compress_struct` with `err` already set.
- **Outputs/Return:** void; `cinfo` is fully initialized.
- **Side effects:** Allocates memory manager state via `jinit_memory_mgr`; zeroes `cinfo`.
- **Calls:** `jinit_memory_mgr`
- **Notes:** Error manager must be set before this call; `input_gamma` is defaulted to `1.0`.

### jpeg_destroy_compress
- **Signature:** `GLOBAL void jpeg_destroy_compress(j_compress_ptr cinfo)`
- **Purpose:** Releases all memory and resources owned by the compression object.
- **Inputs:** `cinfo`
- **Outputs/Return:** void
- **Side effects:** Frees all pooled memory via the common `jpeg_destroy`.
- **Calls:** `jpeg_destroy`

### jpeg_abort_compress
- **Signature:** `GLOBAL void jpeg_abort_compress(j_compress_ptr cinfo)`
- **Purpose:** Aborts an in-progress compression without destroying the object, allowing reuse.
- **Calls:** `jpeg_abort`

### jpeg_suppress_tables
- **Signature:** `GLOBAL void jpeg_suppress_tables(j_compress_ptr cinfo, boolean suppress)`
- **Purpose:** Marks all defined quantization and Huffman tables as already written (`suppress=TRUE`) or not yet written (`suppress=FALSE`), controlling whether `jpeg_start_compress` re-emits them.
- **Inputs:** `cinfo`, `suppress` flag.
- **Side effects:** Mutates `sent_table` on all non-NULL `quant_tbl_ptrs`, `dc_huff_tbl_ptrs`, and `ac_huff_tbl_ptrs` entries.

### jpeg_finish_compress
- **Signature:** `GLOBAL void jpeg_finish_compress(j_compress_ptr cinfo)`
- **Purpose:** Completes compression: terminates the first pass if in scan/raw state, drives remaining multi-pass iterations via the coefficient controller, writes the EOI marker, and terminates the destination.
- **Inputs:** `cinfo` in state `CSTATE_SCANNING`, `CSTATE_RAW_OK`, or `CSTATE_WRCOEFS`.
- **Side effects:** Writes to the output destination; calls `jpeg_abort` to release image-pool memory and reset state.
- **Calls:** `cinfo->master->finish_pass`, `cinfo->master->prepare_for_pass`, `cinfo->coef->compress_data`, `cinfo->progress->progress_monitor`, `cinfo->marker->write_file_trailer`, `cinfo->dest->term_destination`, `jpeg_abort`
- **Notes:** Errors on `JERR_TOO_LITTLE_DATA` if not all scanlines were supplied; errors on `JERR_CANT_SUSPEND` if the coefficient controller suspends unexpectedly.

### jpeg_write_marker
- **Signature:** `GLOBAL void jpeg_write_marker(j_compress_ptr cinfo, int marker, const JOCTET *dataptr, unsigned int datalen)`
- **Purpose:** Writes a raw JPEG marker (e.g., COM, APPn) into the output stream; must be called after `jpeg_start_compress` and before the first scanline.
- **Inputs:** `cinfo`, `marker` code, `dataptr`/`datalen` for marker payload.
- **Side effects:** Writes marker bytes to the output destination.
- **Calls:** `cinfo->marker->write_any_marker`
- **Notes:** Enforces `next_scanline == 0` and valid global state; errors on `JERR_BAD_STATE` otherwise.

### jpeg_write_tables
- **Signature:** `GLOBAL void jpeg_write_tables(j_compress_ptr cinfo)`
- **Purpose:** Writes an abbreviated JPEG file containing only table data (no image), then cleans up; sets all tables as sent (same effect as `jpeg_suppress_tables(..., TRUE)`).
- **Inputs:** `cinfo` in `CSTATE_START`.
- **Side effects:** Initializes destination and marker writer, writes tables to output, calls `jpeg_abort` for cleanup.
- **Calls:** `cinfo->err->reset_error_mgr`, `cinfo->dest->init_destination`, `jinit_marker_writer`, `cinfo->marker->write_tables_only`, `cinfo->dest->term_destination`, `jpeg_abort`

## Control Flow Notes
This file contains lifecycle entry points called by the application layer. `jpeg_create_compress` is the init phase; `jpeg_finish_compress` drives the final multi-pass loop and shutdown. It does not participate in the per-scanline frame loop (that is handled by `jcapistd.c` / `jpeg_write_scanlines`).

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMZERO`, `SIZEOF`)
- `jpeglib.h` / `jpegint.h` — all JPEG types, struct definitions, error codes
- `jinit_memory_mgr` — defined in `jmemmgr.c`
- `jinit_marker_writer` — defined in `jcmarker.c`
- `jpeg_abort`, `jpeg_destroy` — defined in `jcomapi.c`
- All `cinfo->master`, `cinfo->coef`, `cinfo->marker`, `cinfo->dest`, `cinfo->progress` method pointers — implemented in their respective submodule files
