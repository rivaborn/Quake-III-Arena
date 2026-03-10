# code/jpeg-6/jcinit.c

## File Purpose
Performs master module selection and initialization sequencing for the JPEG compressor. It wires together all compression subsystems (preprocessing, DCT, entropy coding, coefficient buffering, etc.) by calling each module's `jinit_*` function in the correct order for a full compression pass.

## Core Responsibilities
- Invoke `jinit_c_master_control` to validate/process compression parameters
- Conditionally initialize preprocessing chain (color conversion, downsampling, prep controller) when raw data input is not used
- Initialize forward DCT module
- Select and initialize the correct entropy encoder (Huffman sequential, Huffman progressive, or error on arithmetic)
- Initialize coefficient and main controllers with appropriate buffering modes
- Initialize the JFIF/JPEG marker writer
- Trigger virtual array allocation via the memory manager
- Write the SOI (Start of Image) file header marker immediately

## Key Types / Data Structures
None defined in this file. Uses types from `jpeglib.h`:

| Name | Kind | Purpose |
|------|------|---------|
| `j_compress_ptr` | typedef (pointer to struct) | Points to the master compression state object (`jpeg_compress_struct`) |

## Global / File-Static State
None.

## Key Functions / Methods

### jinit_compress_master
- **Signature:** `GLOBAL void jinit_compress_master(j_compress_ptr cinfo)`
- **Purpose:** Orchestrates initialization of all JPEG compression submodules for a single image compression run.
- **Inputs:** `cinfo` — fully populated compress struct with image/parameter fields set by the caller.
- **Outputs/Return:** `void`
- **Side effects:** Mutates `cinfo` by populating its `master`, `main`, `prep`, `coef`, `marker`, `cconvert`, `downsample`, `fdct`, and `entropy` subobject pointers. Allocates virtual arrays via the memory manager. Writes the SOI marker to the output destination (`cinfo->dest`).
- **Calls:**
  - `jinit_c_master_control(cinfo, FALSE)`
  - `jinit_color_converter(cinfo)` *(conditional: `!cinfo->raw_data_in`)*
  - `jinit_downsampler(cinfo)` *(conditional)*
  - `jinit_c_prep_controller(cinfo, FALSE)` *(conditional)*
  - `jinit_forward_dct(cinfo)`
  - `ERREXIT(cinfo, JERR_ARITH_NOTIMPL)` *(if `cinfo->arith_code`)*
  - `jinit_phuff_encoder(cinfo)` *(if progressive, guarded by `C_PROGRESSIVE_SUPPORTED`)*
  - `jinit_huff_encoder(cinfo)` *(default sequential path)*
  - `jinit_c_coef_controller(cinfo, <multi-pass flag>)`
  - `jinit_c_main_controller(cinfo, FALSE)`
  - `jinit_marker_writer(cinfo)`
  - `(*cinfo->mem->realize_virt_arrays)((j_common_ptr) cinfo)`
  - `(*cinfo->marker->write_file_header)(cinfo)`
- **Notes:**
  - Arithmetic coding is intentionally unimplemented (`JERR_ARITH_NOTIMPL`).
  - Progressive Huffman encoding is compile-time optional (`#ifdef C_PROGRESSIVE_SUPPORTED`).
  - The coefficient controller is given a full-image buffer only when `num_scans > 1` (multi-scan script) or `optimize_coding` is set — both require two-pass processing.
  - The SOI marker is written here (not deferred) specifically to allow the application to insert custom markers after SOI before frame/scan headers are emitted.

## Control Flow Notes
This file is called **once per image**, during the compression startup phase — specifically from `jpeg_start_compress()` in `jcapistd.c`. It runs entirely before any scanline data is processed. All submodule `jinit_*` calls complete synchronously; actual image data processing begins after this function returns.

## External Dependencies
- `jinclude.h` — platform portability macros, system headers
- `jpeglib.h` — public JPEG API types; also pulls in `jpegint.h` and `jerror.h` (via `JPEG_INTERNALS`)
- **Defined elsewhere (called but not defined here):**
  - `jinit_c_master_control`, `jinit_color_converter`, `jinit_downsampler`, `jinit_c_prep_controller` (`jcmaster.c`, `jccolor.c`, `jcsample.c`, `jcprepct.c`)
  - `jinit_forward_dct` (`jcdctmgr.c`)
  - `jinit_huff_encoder`, `jinit_phuff_encoder` (`jchuff.c`, `jcphuff.c`)
  - `jinit_c_coef_controller`, `jinit_c_main_controller` (`jccoefct.c`, `jcmainct.c`)
  - `jinit_marker_writer` (`jcmarker.c`)
  - `ERREXIT` macro — defined in `jerror.h`
