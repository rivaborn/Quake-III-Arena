# code/jpeg-6/jpegint.h

## File Purpose
Internal header for the Independent JPEG Group's libjpeg-6 library, declaring the vtable-style module interfaces and initialization entry points used to wire together the JPEG compression and decompression pipelines. It is not intended for application-level inclusion.

## Core Responsibilities
- Define the `J_BUF_MODE` enum controlling pass-through vs. full-image buffering modes
- Declare global state machine constants (`CSTATE_*`, `DSTATE_*`) for compress/decompress lifecycle tracking
- Provide virtual dispatch structs (function-pointer tables) for every compression and decompression sub-module
- Declare all `jinit_*` module initializer prototypes that wire up the pipeline at startup
- Declare utility function prototypes (`jdiv_round_up`, `jcopy_sample_rows`, etc.)
- Define portable `RIGHT_SHIFT` macro with optional unsigned-shift workaround
- Provide short-name aliases (`NEED_SHORT_EXTERNAL_NAMES`) for linkers with symbol-length limits

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `J_BUF_MODE` | enum | Selects strip-wise pass-through vs. full-image save/crank buffering for a pipeline pass |
| `jpeg_comp_master` | struct | Vtable for compression master controller; drives multi-pass sequencing |
| `jpeg_c_main_controller` | struct | Vtable for compression main buffer (downsampled data) |
| `jpeg_c_prep_controller` | struct | Vtable for compression pre-processor / downsampling input buffer |
| `jpeg_c_coef_controller` | struct | Vtable for compression coefficient buffer |
| `jpeg_color_converter` | struct | Vtable for RGB→YCbCr (or other) colorspace conversion during compression |
| `jpeg_downsampler` | struct | Vtable for chroma downsampling; exposes `need_context_rows` flag |
| `jpeg_forward_dct` | struct | Vtable for forward DCT + quantization |
| `jpeg_entropy_encoder` | struct | Vtable for Huffman/arithmetic entropy encoding |
| `jpeg_marker_writer` | struct | Vtable for writing JFIF/EXIF markers to the output stream |
| `jpeg_decomp_master` | struct | Vtable for decompression master controller |
| `jpeg_input_controller` | struct | Vtable for input consumption; exposes `has_multiple_scans`, `eoi_reached` |
| `jpeg_d_main_controller` | struct | Vtable for decompression main buffer |
| `jpeg_d_coef_controller` | struct | Vtable for decompression coefficient buffer; exposes `coef_arrays` |
| `jpeg_d_post_controller` | struct | Vtable for post-processing / color quantization buffer |
| `jpeg_marker_reader` | struct | Vtable + state for JPEG marker parsing; exposes `saw_SOI`, `saw_SOF`, restart tracking |
| `jpeg_entropy_decoder` | struct | Vtable for entropy decoding |
| `jpeg_inverse_dct` | struct | Vtable for inverse DCT; per-component `inverse_DCT` function pointer array |
| `jpeg_upsampler` | struct | Vtable for chroma upsampling + color conversion dispatch |
| `jpeg_color_deconverter` | struct | Vtable for YCbCr→RGB (or other) colorspace conversion during decompression |
| `jpeg_color_quantizer` | struct | Vtable for 1-pass or 2-pass color quantization |
| `inverse_DCT_method_ptr` | typedef (function ptr) | Per-component IDCT method signature |

## Global / File-Static State

None. (State constants are `#define` macros, not variables.)

## Key Functions / Methods

No function implementations are present — this is a header. All declarations are prototypes only.

### Notes
- All `jinit_*` functions are compression or decompression sub-module initializers; they allocate and attach the corresponding vtable struct to the `cinfo` object.
- Utility helpers declared: `jdiv_round_up`, `jround_up`, `jcopy_sample_rows`, `jcopy_block_row`, `jzero_far`.
- Constant tables declared: `jpeg_zigzag_order[]`, `jpeg_natural_order[]` (defined in `jutils.c`).

## Control Flow Notes
These structs act as the plugin/vtable layer of the JPEG pipeline. During `jinit_compress_master` / `jinit_master_decompress`, each sub-module's `jinit_*` function is called in sequence to populate the corresponding pointer fields of `j_compress_struct` / `j_decompress_struct`. Afterward, the engine drives processing by invoking `start_pass` → per-row/MCU processing methods → `finish_pass` on each module in order. The `CSTATE_*` / `DSTATE_*` constants track which phase the state machine is in and are checked to guard against out-of-order API calls.

## External Dependencies
- `jpeglib.h` — defines `j_compress_ptr`, `j_decompress_ptr`, `JSAMPARRAY`, `JBLOCKROW`, `JDIMENSION`, `jvirt_barray_ptr`, `jpeg_component_info`, `jpeg_marker_parser_method`, `JMETHOD`, `JPP`, `EXTERN`
- `jmorecfg.h` (transitively) — `INT32`, `boolean`, `FAR`, `MAX_COMPONENTS`
- Symbols defined elsewhere: all `jinit_*` bodies live in their respective `.c` modules (`jcmaster.c`, `jdmaster.c`, `jdmarker.c`, etc.); utility bodies in `jutils.c`
