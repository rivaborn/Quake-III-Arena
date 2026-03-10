# code/jpeg-6/jpeglib.h

## File Purpose
This is the primary public API header for the Independent JPEG Group's (IJG) JPEG library version 6, bundled with Quake III Arena for texture/image decoding. It defines all data structures, type aliases, and function prototypes required by any application that compresses or decompresses JPEG images.

## Core Responsibilities
- Define the master compression (`jpeg_compress_struct`) and decompression (`jpeg_decompress_struct`) context objects
- Declare all public API entry points for the JPEG encode/decode pipeline
- Define supporting data types: quantization tables, Huffman tables, component descriptors, scan scripts
- Declare pluggable manager interfaces (error, memory, progress, source, destination)
- Provide JPEG standard constants (DCT block size, table counts, marker codes)
- Conditionally include internal headers (`jpegint.h`, `jerror.h`) when `JPEG_INTERNALS` is defined

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `jpeg_compress_struct` | struct | Master state for a compression session; holds all parameters, component info, and submodule pointers |
| `jpeg_decompress_struct` | struct | Master state for a decompression session; holds image metadata, output parameters, and submodule pointers |
| `jpeg_common_struct` | struct | Shared base fields (`err`, `mem`, `progress`, `is_decompressor`, `global_state`) between both master structs |
| `jpeg_component_info` | struct | Per color-channel descriptor: sampling factors, DCT block dimensions, table selectors |
| `jpeg_scan_info` | struct | Describes one scan in a multi-scan progressive JPEG file |
| `JQUANT_TBL` | struct | DCT coefficient quantization table (64 values in zigzag order) |
| `JHUFF_TBL` | struct | Huffman coding table (bit-length counts + symbol values) |
| `jpeg_error_mgr` | struct | Pluggable error handler with function pointers for `error_exit`, `emit_message`, `output_message`, `format_message` |
| `jpeg_memory_mgr` | struct | Pluggable pool-based memory manager; supports small/large/virtual-array allocations |
| `jpeg_source_mgr` | struct | Pluggable compressed-data source (buffer pointer + fill/skip/resync callbacks) |
| `jpeg_destination_mgr` | struct | Pluggable compressed-data sink (buffer pointer + flush callbacks) |
| `jpeg_progress_mgr` | struct | Optional progress monitor with pass/row counters |
| `J_COLOR_SPACE` | enum | Known colorspaces: `JCS_UNKNOWN`, `JCS_GRAYSCALE`, `JCS_RGB`, `JCS_YCbCr`, `JCS_CMYK`, `JCS_YCCK` |
| `J_DCT_METHOD` | enum | DCT algorithm selector: `JDCT_ISLOW`, `JDCT_IFAST`, `JDCT_FLOAT` |
| `J_DITHER_MODE` | enum | Output dithering mode for color-quantized decompression |
| `JSAMPROW/JSAMPARRAY/JSAMPIMAGE` | typedef | 1-D/2-D/3-D pixel sample array types |
| `JBLOCK/JBLOCKROW/JBLOCKARRAY` | typedef | DCT coefficient block array types |

## Global / File-Static State

None. This is a pure header file.

## Key Functions / Methods

This file contains only declarations (no definitions). The declared public API functions are:

### jpeg_std_error
- Signature: `struct jpeg_error_mgr *jpeg_std_error(struct jpeg_error_mgr *err)`
- Purpose: Initialize an error manager struct with default handlers.
- Inputs: Pointer to caller-allocated `jpeg_error_mgr`.
- Outputs/Return: The same pointer, initialized.
- Side effects: None.

### jpeg_create_compress / jpeg_create_decompress
- Signature: `void jpeg_create_compress(j_compress_ptr cinfo)` / `void jpeg_create_decompress(j_decompress_ptr cinfo)`
- Purpose: Allocate and initialize a JPEG codec instance. Error manager must be set up beforehand.
- Side effects: Allocates internal memory pools via `cinfo->mem`.

### jpeg_start_compress / jpeg_write_scanlines / jpeg_finish_compress
- Purpose: The three-phase compression pipeline entry points — begin encoding, feed rows, finalize output.
- Notes: `next_scanline` in `cinfo` tracks row progress.

### jpeg_read_header / jpeg_start_decompress / jpeg_read_scanlines / jpeg_finish_decompress
- Purpose: The four-phase decompression pipeline — parse headers, begin decoding, read output rows, finalize.
- Notes: `jpeg_read_header` returns `JPEG_HEADER_OK`, `JPEG_SUSPENDED`, or `JPEG_HEADER_TABLES_ONLY`.

### jpeg_destroy / jpeg_abort
- Purpose: Release all memory and resources for either a compress or decompress instance (generic versions).
- Side effects: Frees all memory pools; object is unusable afterward (`destroy`) or reset (`abort`).

### jpeg_read_coefficients / jpeg_write_coefficients
- Purpose: Raw DCT coefficient access for lossless transcoding.
- Outputs/Return: `jpeg_read_coefficients` returns `jvirt_barray_ptr*` array.

## Control Flow Notes

This file is not directly part of the engine's frame loop. In Quake III, it is consumed during **init/load** time when the renderer (`tr_image.c`) decodes JPEG textures. The typical call sequence is:

1. `jpeg_create_decompress` → `jpeg_stdio_src` (or custom source) → `jpeg_read_header`
2. Set output parameters → `jpeg_start_decompress`
3. Loop: `jpeg_read_scanlines` until `output_scanline >= output_height`
4. `jpeg_finish_decompress` → `jpeg_destroy_decompress`

## External Dependencies

- `code/jpeg-6/jconfig.h` — Platform/compiler configuration flags (`HAVE_PROTOTYPES`, `HAVE_UNSIGNED_CHAR`, `JDCT_DEFAULT`, etc.)
- `code/jpeg-6/jmorecfg.h` — Type definitions (`JSAMPLE`, `JCOEF`, `JOCTET`, `UINT8`, `UINT16`, `INT32`, `JDIMENSION`), linkage macros (`EXTERN`, `METHODDEF`, `LOCAL`)
- `code/jpeg-6/jpegint.h` — Internal submodule struct definitions (included only when `JPEG_INTERNALS` is defined)
- `code/jpeg-6/jerror.h` — Error/message code enum and `ERREXIT`/`WARNMS`/`TRACEMS` macros (included only when `JPEG_INTERNALS` is defined)
- All internal submodule structs (`jpeg_comp_master`, `jpeg_entropy_encoder`, `jpeg_inverse_dct`, etc.) are defined elsewhere (in `jpegint.h`) and referenced here only as forward-declared pointers
