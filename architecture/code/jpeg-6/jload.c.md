# code/jpeg-6/jload.c

## File Purpose
Provides a single function to load a JPEG image from the Quake III filesystem into a heap-allocated pixel buffer. It bridges the engine's virtual filesystem and memory allocator with the libjpeg-6 decompression API.

## Core Responsibilities
- Open a JPEG file via the engine's `FS_FOpenFileRead` filesystem abstraction
- Initialize and configure a libjpeg decompression object with standard error handling
- Drive the full libjpeg decompression pipeline (header read → start → scanline loop → finish → destroy)
- Allocate output pixel memory via engine's `Z_Malloc` (zone allocator)
- Return image dimensions and pixel data to the caller via out-parameters
- Close the file handle with `FS_FCloseFile` after decompression

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `jpeg_decompress_struct` (cinfo) | struct (libjpeg) | Holds all JPEG decompression state and parameters |
| `jpeg_error_mgr` (jerr) | struct (libjpeg) | Standard error handler; calls `exit()` on fatal JPEG errors |
| `fileHandle_t` | typedef (int) | Engine-opaque file handle used with FS_* calls |
| `JSAMPARRAY` | typedef (libjpeg) | Pointer to array of scanline row pointers |

## Global / File-Static State
None.

## Key Functions / Methods

### LoadJPG
- **Signature:** `int LoadJPG( const char *filename, unsigned char **pic, int *width, int *height )`
- **Purpose:** Fully decodes a JPEG file from the virtual filesystem into a flat RGBA/RGB pixel buffer.
- **Inputs:**
  - `filename` — VFS-relative path to the JPEG file
  - `pic` — out-param; receives pointer to allocated pixel buffer
  - `width`, `height` — out-params; receive decoded image dimensions
- **Outputs/Return:** Returns `1` on success, `0` if the file could not be opened. `*pic` is set to a `Z_Malloc`-allocated buffer on success.
- **Side effects:**
  - Allocates heap memory via `Z_Malloc` (engine zone allocator); caller is responsible for freeing
  - Opens and closes a VFS file handle via `FS_FOpenFileRead` / `FS_FCloseFile`
  - libjpeg may call `exit()` on fatal decode errors (standard error handler, no `setjmp` recovery override)
- **Calls:** `FS_FOpenFileRead`, `jpeg_std_error`, `jpeg_create_decompress`, `jpeg_stdio_src`, `jpeg_read_header`, `jpeg_start_decompress`, `Z_Malloc`, `jpeg_read_scanlines`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`, `FS_FCloseFile`
- **Notes:**
  - **Bug:** The scanline read loop passes `buffer` as `(JSAMPARRAY)out + (row_stride * cinfo.output_scanline)`. `out` is `unsigned char *`; casting it to `JSAMPARRAY` before arithmetic treats a byte pointer as a pointer-to-pointer, producing wrong addresses. The correct approach is to pass `&out[row_stride * cinfo.output_scanline]` cast to `JSAMPARRAY`, or use a temporary `JSAMPROW`.
  - No `setjmp`/`longjmp` error recovery is installed; a corrupt JPEG causes `exit()` via the default error handler.
  - `jpeg_stdio_src` is called with a `fileHandle_t` (integer), not a `FILE *`; this works only because the local `jpeglib.h` redefines `jpeg_stdio_src` to accept `unsigned char *`, which in this codebase aliases the engine's file handle.

## Control Flow Notes
This file is a load-time utility, not part of the frame loop. It is called during texture/image loading (e.g., from `tr_image.c` in the renderer) when a `.jpg` asset is needed. There is no init or shutdown involvement; each call is self-contained.

## External Dependencies
- `../game/q_shared.h` — `fileHandle_t`, `qboolean`, `Z_Malloc`
- `../qcommon/qcommon.h` — `FS_FOpenFileRead`, `FS_FCloseFile`
- `jpeglib.h` (local jpeg-6) — entire libjpeg decompression API
- **Defined elsewhere:** `FS_FOpenFileRead`, `FS_FCloseFile` (filesystem module); `Z_Malloc` (zone memory, `qcommon/`); all `jpeg_*` functions (libjpeg-6 implementation files in `code/jpeg-6/`)
