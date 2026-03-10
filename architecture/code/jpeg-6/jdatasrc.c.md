# code/jpeg-6/jdatasrc.c

## File Purpose
Implements a JPEG decompression data source manager that reads compressed JPEG data from an in-memory byte buffer (modified from the original stdio-based version). It satisfies the `jpeg_source_mgr` interface required by the IJG JPEG library.

## Core Responsibilities
- Provide a concrete `jpeg_source_mgr` implementation for memory-backed JPEG input
- Initialize and manage a fixed-size intermediate read buffer (`INPUT_BUF_SIZE = 4096`)
- Refill the decompressor's input buffer by copying from the in-memory source pointer
- Support skipping over unneeded data segments (APPn markers, etc.)
- Register all source manager callbacks on the `j_decompress_ptr` object via `jpeg_stdio_src`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_source_mgr` | struct | Extends `jpeg_source_mgr` with a raw memory pointer (`infile`) and an intermediate `JOCTET` buffer; serves as the concrete source object |
| `my_src_ptr` | typedef (pointer) | Convenience alias for `my_source_mgr *` |

## Global / File-Static State
None.

## Key Functions / Methods

### init_source
- **Signature:** `METHODDEF void init_source(j_decompress_ptr cinfo)`
- **Purpose:** Resets the `start_of_file` flag before reading a new image; does not clear the buffer.
- **Inputs:** `cinfo` — active decompression context
- **Outputs/Return:** void
- **Side effects:** Sets `src->start_of_file = TRUE`
- **Calls:** None
- **Notes:** Allows sequential reading of multiple JPEG images from the same source without re-initializing the buffer.

### fill_input_buffer
- **Signature:** `METHODDEF boolean fill_input_buffer(j_decompress_ptr cinfo)`
- **Purpose:** Copies the next `INPUT_BUF_SIZE` bytes from the in-memory source into the intermediate buffer and advances the source pointer.
- **Inputs:** `cinfo` — active decompression context
- **Outputs/Return:** Always `TRUE` (no suspension support)
- **Side effects:** Advances `src->infile` by `INPUT_BUF_SIZE`; updates `pub.next_input_byte` and `pub.bytes_in_buffer`; clears `start_of_file`
- **Calls:** `memcpy`
- **Notes:** **Potential overread bug** — unconditionally copies exactly `INPUT_BUF_SIZE` bytes regardless of actual remaining data; no EOF or bounds check. No empty-file or premature-EOF handling (unlike the original stdio version).

### skip_input_data
- **Signature:** `METHODDEF void skip_input_data(j_decompress_ptr cinfo, long num_bytes)`
- **Purpose:** Advances the read position by `num_bytes`, crossing buffer boundaries by calling `fill_input_buffer` as needed.
- **Inputs:** `cinfo`, `num_bytes` — bytes to skip
- **Outputs/Return:** void
- **Side effects:** Modifies `pub.next_input_byte`, `pub.bytes_in_buffer`; may call `fill_input_buffer` multiple times
- **Calls:** `fill_input_buffer`
- **Notes:** Assumes `fill_input_buffer` never returns `FALSE`; no suspension handling.

### term_source
- **Signature:** `METHODDEF void term_source(j_decompress_ptr cinfo)`
- **Purpose:** Cleanup callback invoked by `jpeg_finish_decompress`; no-op here since no file handle needs closing.
- **Inputs:** `cinfo`
- **Outputs/Return:** void
- **Side effects:** None

### jpeg_stdio_src
- **Signature:** `GLOBAL void jpeg_stdio_src(j_decompress_ptr cinfo, unsigned char *infile)`
- **Purpose:** Public entry point; allocates (once, permanently) the `my_source_mgr` and its buffer, then wires all function pointers onto `cinfo->src`.
- **Inputs:** `cinfo`, `infile` — pointer to raw JPEG data in memory
- **Outputs/Return:** void
- **Side effects:** Allocates `my_source_mgr` and a 4096-byte `JOCTET` buffer via JPEG pool allocator (`JPOOL_PERMANENT`); sets all `cinfo->src` method pointers
- **Calls:** `cinfo->mem->alloc_small` (×2), assigns `jpeg_resync_to_restart` (library default)
- **Notes:** Safe to call multiple times on the same `cinfo`; re-allocation is skipped if `cinfo->src != NULL`. The name `jpeg_stdio_src` is retained from the original IJG API signature even though this version operates on a memory buffer, not a stdio `FILE *`.

## Control Flow Notes
Called once during JPEG load setup (before `jpeg_read_header`). The source manager callbacks are then driven internally by the JPEG decompressor: `init_source` → repeated `fill_input_buffer` / `skip_input_data` calls during header parsing and scanline decompression → `term_source` on `jpeg_finish_decompress`. This file has no frame-loop or render involvement.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `memcpy` via `<string.h>`)
- `jpeglib.h` — `jpeg_source_mgr`, `j_decompress_ptr`, `JOCTET`, `JPOOL_PERMANENT`, `SIZEOF`, `jpeg_resync_to_restart`
- `jerror.h` — error/trace macros (included transitively; not directly used in this file)
- **Defined elsewhere:** `jpeg_resync_to_restart` (IJG library default restart-marker recovery)
