# code/jpeg-6/jdatadst.c

## File Purpose
Implements a stdio-based JPEG compression data destination manager for the IJG JPEG library. It provides the output buffering and flushing logic that routes compressed JPEG bytes to a `FILE*` stream during encoding.

## Core Responsibilities
- Allocate and manage a 4096-byte output buffer for compressed JPEG data
- Flush the full buffer to disk via `fwrite` when it fills during compression
- Flush any remaining partial buffer bytes at end-of-compression
- Install the destination manager's three callback functions onto a `j_compress_ptr`
- Reuse an existing destination object if one is already attached to the compressor

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_destination_mgr` | struct (typedef) | Extends `jpeg_destination_mgr` with a `FILE*` and a `JOCTET*` buffer pointer for stdio-based output |
| `my_dest_ptr` | typedef (pointer) | Convenience alias for `my_destination_mgr *` |

## Global / File-Static State
None.

## Key Functions / Methods

### init_destination
- **Signature:** `METHODDEF void init_destination(j_compress_ptr cinfo)`
- **Purpose:** Allocates the output byte buffer from the JPEG image pool and resets the destination manager's pointer and free-count fields.
- **Inputs:** `cinfo` — active compression context
- **Outputs/Return:** void
- **Side effects:** Allocates `OUTPUT_BUF_SIZE` (4096) bytes from `JPOOL_IMAGE`; sets `pub.next_output_byte` and `pub.free_in_buffer`.
- **Calls:** `cinfo->mem->alloc_small`
- **Notes:** Called by `jpeg_start_compress` before any data is written. Buffer lifetime is tied to the image pool.

### empty_output_buffer
- **Signature:** `METHODDEF boolean empty_output_buffer(j_compress_ptr cinfo)`
- **Purpose:** Writes the entire 4096-byte buffer to the output file when it is full, then resets the buffer pointers.
- **Inputs:** `cinfo` — active compression context
- **Outputs/Return:** `TRUE` on success; calls `ERREXIT` on write failure (does not return `FALSE` — suspension is not supported here).
- **Side effects:** I/O write to `dest->outfile`; resets `next_output_byte` and `free_in_buffer`.
- **Calls:** `JFWRITE`, `ERREXIT`
- **Notes:** Always writes the full `OUTPUT_BUF_SIZE` bytes regardless of `next_output_byte`/`free_in_buffer` state, as required by the JPEG library contract for non-suspending destinations.

### term_destination
- **Signature:** `METHODDEF void term_destination(j_compress_ptr cinfo)`
- **Purpose:** Flushes any remaining buffered bytes and calls `fflush`; verifies no file error occurred.
- **Inputs:** `cinfo` — active compression context
- **Outputs/Return:** void
- **Side effects:** I/O write and flush to `dest->outfile`; calls `ERREXIT` on write/flush error.
- **Calls:** `JFWRITE`, `fflush`, `ferror`, `ERREXIT`
- **Notes:** Not called by `jpeg_abort` or `jpeg_destroy`; the caller is responsible for cleanup in error paths.

### jpeg_stdio_dest
- **Signature:** `GLOBAL void jpeg_stdio_dest(j_compress_ptr cinfo, FILE * outfile)`
- **Purpose:** Public entry point that attaches a stdio destination manager to a compressor object. Allocates the `my_destination_mgr` in the permanent pool on first call; on subsequent calls reuses the existing allocation.
- **Inputs:** `cinfo` — compressor; `outfile` — already-open writable `FILE*`
- **Outputs/Return:** void
- **Side effects:** Allocates from `JPOOL_PERMANENT` on first call; sets all three callback pointers and `dest->outfile`.
- **Calls:** `cinfo->mem->alloc_small`
- **Notes:** Permanent-pool allocation means the same destination object survives across multiple JPEG images written to the same file. Caller owns the `FILE*` and must close it after `jpeg_finish_compress`.

## Control Flow Notes
This file is used only during **compression**. The call sequence is:
1. `jpeg_stdio_dest` — setup (before `jpeg_start_compress`)
2. `init_destination` — called internally by `jpeg_start_compress`
3. `empty_output_buffer` — called internally whenever the 4096-byte buffer fills
4. `term_destination` — called internally by `jpeg_finish_compress`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `JFWRITE` macro, `<stdio.h>`
- `jpeglib.h` — `j_compress_ptr`, `jpeg_destination_mgr`, `JOCTET`, `JPOOL_IMAGE`, `JPOOL_PERMANENT`, `boolean`
- `jerror.h` — `ERREXIT`, `JERR_FILE_WRITE`
- `fwrite`, `fflush`, `ferror` — C standard I/O (defined in `<stdio.h>`)
- `jpeg_start_compress`, `jpeg_finish_compress` — defined elsewhere; invoke the callbacks installed here
