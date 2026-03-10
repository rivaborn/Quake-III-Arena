# code/jpeg-6/jdtrans.c

## File Purpose
Implements JPEG transcoding decompression — reading raw DCT coefficient arrays from a JPEG file without performing full image decompression. This is the "lossless" path used when transcoding (e.g., re-compressing with different parameters without quality loss).

## Core Responsibilities
- Provide `jpeg_read_coefficients()`, the public entry point for transcoding decompression
- Drive the input consumption loop to absorb the entire JPEG file into virtual coefficient-block arrays
- Initialize a minimal subset of decompressor modules sufficient for coefficient extraction (no IDCT, upsampling, color conversion, or quantization)
- Select and initialize the correct entropy decoder (Huffman sequential or progressive)
- Allocate and realize the full-image virtual coefficient buffer
- Initialize progress monitoring with scan-count estimates appropriate for transcoding

## Key Types / Data Structures
None defined in this file; all types come from `jpeglib.h`.

| Name | Kind | Purpose |
|---|---|---|
| `j_decompress_ptr` | typedef (pointer to struct) | Handle to the JPEG decompressor state machine |
| `jvirt_barray_ptr` | typedef (pointer to struct) | Descriptor for a virtual DCT coefficient block array |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_read_coefficients
- **Signature:** `GLOBAL jvirt_barray_ptr * jpeg_read_coefficients(j_decompress_ptr cinfo)`
- **Purpose:** Public API to read all DCT coefficients from a JPEG stream into virtual arrays. Caller must have already called `jpeg_read_header()`.
- **Inputs:** `cinfo` — initialized decompressor instance in `DSTATE_READY` or `DSTATE_RDCOEFS` state.
- **Outputs/Return:** Pointer to array of `jvirt_barray_ptr` (one per color component), or `NULL` if suspended.
- **Side effects:** Advances `cinfo->global_state` to `DSTATE_RDCOEFS` on first call, then to `DSTATE_STOPPING` on completion. Calls progress monitor hook. Triggers module initialization via `transdecode_master_selection` on first call.
- **Calls:** `transdecode_master_selection`, `cinfo->progress->progress_monitor`, `cinfo->inputctl->consume_input`, `ERREXIT1`
- **Notes:** Returns `NULL` on suspension (only possible with a suspending data source). On re-entry, skips re-initialization since state is already `DSTATE_RDCOEFS`. Progress counter is ratcheted up if the file has more scans than initially estimated.

### transdecode_master_selection
- **Signature:** `LOCAL void transdecode_master_selection(j_decompress_ptr cinfo)`
- **Purpose:** Minimal decompressor module initialization for transcoding. Replaces the full `jdmaster.c` initialization — only entropy decoding and coefficient buffering are set up; no IDCT, upsampling, color conversion, or post-processing.
- **Inputs:** `cinfo` — decompressor with image parameters populated from `jpeg_read_header()`.
- **Outputs/Return:** void
- **Side effects:** Initializes entropy decoder module (`jinit_huff_decoder` or `jinit_phuff_decoder`). Initializes full-image coefficient controller (`jinit_d_coef_controller` with `TRUE`). Realizes all virtual arrays via `cinfo->mem->realize_virt_arrays`. Starts the first input pass via `cinfo->inputctl->start_input_pass`. Populates `cinfo->progress` fields if a progress monitor is installed.
- **Calls:** `jinit_phuff_decoder`, `jinit_huff_decoder`, `jinit_d_coef_controller`, `cinfo->mem->realize_virt_arrays`, `cinfo->inputctl->start_input_pass`, `ERREXIT`
- **Notes:** Arithmetic coding is explicitly unsupported (`JERR_ARITH_NOTIMPL`). Progressive Huffman requires compile-time `D_PROGRESSIVE_SUPPORTED`. Progress pass limit is estimated: 2 DC + 3×N AC scans for progressive, N scans for multi-scan sequential, 1 for single-scan.

## Control Flow Notes
This file is not part of the normal frame/render loop. It provides an **alternative decompression path** invoked once per JPEG file by a transcoder application:
1. App calls `jpeg_read_header()` (defined elsewhere, `jdapimin.c`)
2. App calls `jpeg_read_coefficients()` — absorbs entire file
3. App passes returned arrays to `jpeg_write_coefficients()` for re-encoding
4. App calls `jpeg_finish_decompress()` to release virtual arrays

The internal loop in `jpeg_read_coefficients` runs `consume_input` repeatedly until `JPEG_REACHED_EOI`, making this a blocking, whole-file operation from the caller's perspective (unless a suspending source is used).

## External Dependencies
- **`jinclude.h`** — platform portability macros (`SIZEOF`, `MEMCOPY`, system headers)
- **`jpeglib.h`** — all public JPEG types and state machine structs; includes `jpegint.h` and `jerror.h` when `JPEG_INTERNALS` is defined
- **Defined elsewhere:**
  - `jinit_huff_decoder` — sequential Huffman decoder init (`jdhuff.c`)
  - `jinit_phuff_decoder` — progressive Huffman decoder init (`jdphuff.c`)
  - `jinit_d_coef_controller` — coefficient buffer controller init (`jdcoefct.c`)
  - `ERREXIT`, `ERREXIT1` — error macros resolving via `cinfo->err->error_exit`
  - `DSTATE_READY`, `DSTATE_RDCOEFS`, `DSTATE_STOPPING` — decompressor state constants (`jpegint.h`)
