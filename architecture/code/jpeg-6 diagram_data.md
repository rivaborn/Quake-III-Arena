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

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMZERO`, `SIZEOF`)
- `jpeglib.h` / `jpegint.h` — all JPEG types, struct definitions, error codes
- `jinit_memory_mgr` — defined in `jmemmgr.c`
- `jinit_marker_writer` — defined in `jcmarker.c`
- `jpeg_abort`, `jpeg_destroy` — defined in `jcomapi.c`
- All `cinfo->master`, `cinfo->coef`, `cinfo->marker`, `cinfo->dest`, `cinfo->progress` method pointers — implemented in their respective submodule files

# code/jpeg-6/jcapistd.c
## File Purpose
Implements the standard JPEG compression API entry points for full-compression workflows: initializing a compression session, writing scanlines of image data, and writing raw downsampled data. Intentionally separated from `jcapimin.c` to prevent linking the full compressor into transcoding-only applications.

## Core Responsibilities
- Initialize a compression session and activate all encoder submodules
- Accept and process scanline-format image data from the caller
- Accept and process pre-downsampled (raw) image data in iMCU-row units
- Track and report scanline progress via an optional progress monitor hook
- Enforce call-sequence validity via `global_state` checks

## External Dependencies
- `jinclude.h` — system header portability layer (`MEMZERO`, `MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` — all public JPEG types and the `jpeg_compress_struct` definition
- `jpegint.h` (via `JPEG_INTERNALS`) — internal submodule interface structs (`jpeg_comp_master`, `jpeg_c_main_controller`, `jpeg_c_coef_controller`, etc.)
- `jerror.h` (via `JPEG_INTERNALS`) — `ERREXIT1`, `WARNMS`, `ERREXIT` macros and error codes
- **Defined elsewhere:** `jinit_compress_master` (jcmaster.c), `jpeg_suppress_tables` (jcparam.c), all vtable method implementations (`process_data`, `compress_data`, `pass_startup`, `prepare_for_pass`, `progress_monitor`, `init_destination`, `reset_error_mgr`)

# code/jpeg-6/jccoefct.c
## File Purpose
Implements the coefficient buffer controller for JPEG compression. It sits between the forward-DCT stage and entropy encoding, managing how DCT coefficient blocks are collected, buffered, and fed to the entropy encoder. It is the top-level controller of the JPEG compressor proper.

## Core Responsibilities
- Initialize and manage the coefficient buffer (single-MCU or full-image virtual arrays)
- Dispatch the correct `compress_data` function pointer based on pass mode
- Run forward DCT on input sample rows and accumulate coefficient blocks into MCUs
- Handle padding (dummy blocks) at right and bottom image edges
- Support single-pass (pass-through) and multi-pass (Huffman optimization / multi-scan) compression
- Suspend and resume mid-row if the entropy encoder stalls

## External Dependencies
- `jinclude.h` — system includes, SIZEOF, MEMZERO macros
- `jpeglib.h` / `jpegint.h` — `j_compress_ptr`, `jpeg_component_info`, `jvirt_barray_ptr`, `JBLOCKROW`, `J_BUF_MODE`, `JDIMENSION`, etc.
- **Defined elsewhere:** `jzero_far`, `jround_up` (utility routines); `cinfo->fdct->forward_DCT` (forward DCT module); `cinfo->entropy->encode_mcu` (entropy encoder); `cinfo->mem->*` (memory manager); `ERREXIT` (error handler macro)

# code/jpeg-6/jccolor.c
## File Purpose
Implements input colorspace conversion for the IJG JPEG compressor. It transforms application-supplied pixel data (RGB, CMYK, grayscale, YCbCr, YCCK) into the JPEG internal colorspace before encoding. This is the compression-side counterpart to `jdcolor.c`.

## Core Responsibilities
- Allocate and initialize lookup tables for fixed-point RGB→YCbCr conversion
- Convert interleaved RGB input rows to planar YCbCr output (most common path)
- Convert RGB rows to grayscale (Y-only)
- Convert CMYK rows to YCCK (inverts CMY, passes K through)
- Pass through grayscale or multi-component data unchanged (`null_convert`, `grayscale_convert`)
- Select and wire the correct conversion function pointer pair (`start_pass` + `color_convert`) during module initialization

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `JSAMPLE*` types, `jpeg_color_converter`, `J_COLOR_SPACE` enum, `JPOOL_IMAGE`, `ERREXIT`, `METHODDEF`, `GLOBAL`
- `jmorecfg.h` (via jpeglib) — `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE`, `INT32`
- `alloc_small` — defined in the JPEG memory manager (`jmemmgr.c`), called through `cinfo->mem`

# code/jpeg-6/jcdctmgr.c
## File Purpose
Manages the forward DCT (Discrete Cosine Transform) pipeline for JPEG compression. It selects the appropriate DCT algorithm at initialization, precomputes scaled quantization divisor tables per component, and drives the encode-time DCT-and-quantize step for each 8×8 sample block.

## Core Responsibilities
- Allocate and initialize the `my_fdct_controller` subobject and wire it into `cinfo->fdct`
- Select the active DCT routine (`jpeg_fdct_islow`, `jpeg_fdct_ifast`, or `jpeg_fdct_float`) based on `cinfo->dct_method`
- Precompute per-quantization-table divisor arrays (scaled and reordered from zigzag) during `start_pass`
- Load 8×8 pixel blocks into a workspace with unsigned-to-signed bias removal
- Invoke the chosen DCT routine in-place on the workspace
- Quantize/descale the 64 DCT coefficients and write them to the output coefficient block array

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, `JBLOCKROW`, `JSAMPARRAY`, `JCOEF`, `NUM_QUANT_TBLS`, `DCTSIZE2`, `JPOOL_IMAGE`
- `jdct.h` — `DCTELEM`, `forward_DCT_method_ptr`, `float_DCT_method_ptr`, `FAST_FLOAT`, fixed-point macros (`DESCALE`, `MULTIPLY16V16`, `SHIFT_TEMPS`)
- **Defined elsewhere:** `jpeg_fdct_islow` (`jfdctint.c`), `jpeg_fdct_ifast` (`jfdctfst.c`), `jpeg_fdct_float` (`jfdctflt.c`), `jpeg_zigzag_order` (IJG internal table), `ERREXIT`/`ERREXIT1` (error handler macros from `jerror.h`)

# code/jpeg-6/jchuff.c
## File Purpose
Implements Huffman entropy encoding for the IJG JPEG compression library. It handles both standard encoding (writing coded bits to the output stream) and a statistics-gathering pass used to generate optimal Huffman tables.

## Core Responsibilities
- Initialize and configure the Huffman encoder for a compression scan
- Build derived lookup tables (`c_derived_tbl`) from raw JHUFF_TBL definitions
- Emit variable-length Huffman codes and raw coefficient bits to the output buffer
- Encode one MCU's worth of DCT coefficient blocks (DC + AC) per JPEG Section F.1.2
- Handle output suspension and MCU-level rollback via `savable_state`
- Emit restart markers and reset DC predictions at restart boundaries
- Gather symbol frequency statistics and generate optimal Huffman tables (when `ENTROPY_OPT_SUPPORTED`)

## External Dependencies
- `jinclude.h` — portability macros (`MEMZERO`, `MEMCOPY`, `SIZEOF`)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_entropy_encoder`, `JHUFF_TBL`, `jpeg_component_info`, `jpeg_destination_mgr`
- `jchuff.h` — `c_derived_tbl`, shared extern declarations for `jpeg_make_c_derived_tbl` / `jpeg_gen_optimal_table`
- **Defined elsewhere**: `jpeg_natural_order` (zigzag index table, `jpegint.h`/`jutils.c`), `jpeg_alloc_huff_table`, all JPEG error codes (`JERR_*`), `ERREXIT`/`ERREXIT1` macros

# code/jpeg-6/jchuff.h
## File Purpose
Declares shared data structures and function prototypes for JPEG Huffman entropy encoding, used by both the sequential encoder (`jchuff.c`) and the progressive encoder (`jcphuff.c`).

## Core Responsibilities
- Define the `c_derived_tbl` structure representing a pre-computed Huffman encoding table
- Declare `jpeg_make_c_derived_tbl` for expanding a raw Huffman table into derived (ready-to-use) form
- Declare `jpeg_gen_optimal_table` for generating an optimal Huffman table from symbol frequency data
- Provide short-name aliases for linkers with limited external symbol length support

## External Dependencies
- `jpeglib.h` / `jpegint.h` — defines `j_compress_ptr`, `JHUFF_TBL`, `JPP`, `EXTERN`
- `jpeg_make_c_derived_tbl`, `jpeg_gen_optimal_table` — defined in `jchuff.c`

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

# code/jpeg-6/jcmainct.c
## File Purpose
Implements the main buffer controller for the JPEG compression pipeline. It sits between the pre-processor (downsampling/color conversion) and the DCT/entropy coefficient compressor, managing the intermediate strip buffer of downsampled JPEG-colorspace data.

## Core Responsibilities
- Allocate and manage per-component strip buffers (or optional full-image virtual arrays) to hold downsampled data
- Initialize pass state (iMCU row counters, buffer mode) at the start of each compression pass
- Drive the data flow loop: pull rows from the preprocessor into the strip buffer, then push complete iMCU rows to the coefficient compressor
- Handle compressor suspension (output-not-consumed) by backing up the input row counter and retrying on the next call
- Expose the `start_pass` and `process_data` method pointers on `jpeg_c_main_controller`

## External Dependencies
- `jinclude.h` — system includes, `MEMZERO`/`MEMCOPY`, `SIZEOF`
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `jpeg_c_main_controller`, `jpeg_component_info`, `JDIMENSION`, `JSAMPARRAY`, `J_BUF_MODE`, `DCTSIZE`, `MAX_COMPONENTS`, `jround_up`
- `ERREXIT` — error macro defined in `jerror.h`
- `cinfo->prep->pre_process_data` — defined in `jcprepct.c`
- `cinfo->coef->compress_data` — defined in `jccoefct.c`
- `cinfo->mem->alloc_small`, `alloc_sarray`, `request_virt_sarray`, `access_virt_sarray` — defined in `jmemmgr.c`

# code/jpeg-6/jcmarker.c
## File Purpose
Implements the JPEG marker writer module for the IJG JPEG compression library. It serializes all required JPEG datastream markers (SOI, SOF, SOS, DHT, DQT, DRI, APP0, APP14, EOI, etc.) to the output destination buffer.

## Core Responsibilities
- Emit raw bytes and 2-byte big-endian integers to the output destination
- Write quantization table markers (DQT) and Huffman table markers (DHT)
- Write frame header (SOFn) and scan header (SOS, DRI) markers
- Write file header (SOI + optional JFIF APP0 / Adobe APP14) and trailer (EOI)
- Write abbreviated table-only datastreams
- Initialize the `jpeg_marker_writer` vtable on `cinfo->marker`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMCOPY`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `j_compress_ptr`, `jpeg_marker_writer`, `JQUANT_TBL`, `JHUFF_TBL`, `jpeg_component_info`, `jpeg_destination_mgr`
- `jerror.h` — `ERREXIT`, `ERREXIT1`, `TRACEMS` macros (defined elsewhere via error manager)
- `C_ARITH_CODING_SUPPORTED` — conditional compile guard for `emit_dac` body (defined in `jconfig.h`)

# code/jpeg-6/jcmaster.c
## File Purpose
Implements the master control logic for the IJG JPEG compressor. It handles parameter validation, initial image geometry setup, multi-scan script validation, and inter-pass sequencing (determining pass types and ordering for single-pass, Huffman-optimization, and multi-scan progressive compression).

## Core Responsibilities
- Validate image dimensions, sampling factors, and component counts before compression begins
- Compute per-component DCT block dimensions, downsampled sizes, and MCU layout
- Validate multi-scan scripts (including progressive JPEG spectral/successive-approximation parameters)
- Set up scan parameters and MCU geometry for each scan
- Drive the pass pipeline: dispatch `start_pass` calls to all active submodules in the correct order
- Track pass number, scan number, and pass type state across the full compression sequence
- Initialize and wire up the `jpeg_comp_master` vtable on the `cinfo` object

## External Dependencies
- `jinclude.h` — system include resolution, `SIZEOF`, `MEMCOPY` macros
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — all JPEG struct definitions, `JMETHOD`, `JPOOL_IMAGE`, `DCTSIZE`, `DCTSIZE2`, `MAX_COMPONENTS`, `MAX_COMPS_IN_SCAN`, `C_MAX_BLOCKS_IN_MCU`
- `jerror.h` — `ERREXIT`, `ERREXIT1`, `ERREXIT2` macros and error codes
- `jdiv_round_up` — defined elsewhere (jutils.c); integer ceiling division
- All submodule vtable objects (`cconvert`, `downsample`, `prep`, `fdct`, `entropy`, `coef`, `main`, `marker`) — defined and initialized in their respective source files

# code/jpeg-6/jcomapi.c
## File Purpose
Provides the shared application interface routines for the IJG JPEG library that are common to both compression and decompression paths. It implements object lifecycle management (abort and destroy) and convenience allocators for quantization and Huffman tables.

## Core Responsibilities
- Abort an in-progress JPEG operation without destroying the object, resetting it for reuse
- Fully destroy a JPEG object and release all associated memory
- Allocate and zero-initialize quantization table (`JQUANT_TBL`) instances
- Allocate and zero-initialize Huffman table (`JHUFF_TBL`) instances

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, system headers)
- `jpeglib.h` — defines `j_common_ptr`, `JQUANT_TBL`, `JHUFF_TBL`, `jpeg_memory_mgr`, pool constants (`JPOOL_PERMANENT`, `JPOOL_NUMPOOLS`), state constants (`DSTATE_START`, `CSTATE_START`)
- `jpeg_memory_mgr::free_pool`, `::self_destruct`, `::alloc_small` — defined elsewhere (implemented in `jmemmgr.c`)

# code/jpeg-6/jconfig.h
## File Purpose
Platform-specific configuration header for the JPEG-6 library, targeting Watcom C/C++ on MS-DOS or OS/2. It defines compiler/platform capability macros consumed by the rest of the libjpeg source tree.

## Core Responsibilities
- Advertises C language feature availability (prototypes, unsigned types, stddef/stdlib headers)
- Configures pointer model and string library preferences for the target platform
- Selects the default and fastest DCT (Discrete Cosine Transform) algorithm variant
- Conditionally enables supported image file formats for the standalone cjpeg/djpeg tools
- Guards internal-only settings (shift behavior) behind `JPEG_INTERNALS`

## External Dependencies
- No includes. Intended to be the first platform-adaptation header consumed by `jinclude.h`.
- `JDCT_FLOAT` — enum value defined in `jpeglib.h`; referenced here before that header is included, so order of inclusion matters.


# code/jpeg-6/jcparam.c
## File Purpose
Provides optional default-setting and parameter-configuration routines for the IJG JPEG compressor. Applications call these helpers to configure quantization tables, Huffman tables, colorspace, and encoding options before starting compression.

## Core Responsibilities
- Build and install scaled quantization tables from standard JPEG spec templates
- Convert user-friendly quality ratings (0–100) to quantization scale factors
- Install standard Huffman tables (DC/AC, luma/chroma) per JPEG spec section K.3
- Set all compressor defaults (quality 75, Huffman coding, no restart markers, etc.)
- Map input colorspace to JPEG output colorspace and configure per-component sampling
- Optionally generate a progressive JPEG scan script

## External Dependencies
- `jinclude.h` — system includes, `MEMCOPY`, `SIZEOF`, platform warning suppression
- `jpeglib.h` / `jpegint.h` / `jerror.h` (via `JPEG_INTERNALS`) — all struct definitions, constants, error macros
- **Defined elsewhere:** `jpeg_alloc_quant_table`, `jpeg_alloc_huff_table` (memory module); `ERREXIT`/`ERREXIT1`/`ERREXIT2` (error handler macros); `CSTATE_START`, `JPOOL_PERMANENT`, `BITS_IN_JSAMPLE`, `DCTSIZE2`, `MAX_COMPONENTS`, `NUM_ARITH_TBLS` (constants from jpegint.h/jconfig.h)

# code/jpeg-6/jcphuff.c
## File Purpose
Implements Huffman entropy encoding for progressive JPEG compression, handling all four scan types: DC initial, DC refinement, AC initial, and AC refinement passes. This is the progressive counterpart to the sequential Huffman encoder in `jchuff.c`.

## Core Responsibilities
- Initialize and configure the progressive entropy encoder per scan type
- Encode DC coefficient initial scans with point-transform and differential coding
- Encode AC coefficient initial scans with run-length and EOB-run coding
- Encode DC refinement scans (single bit per coefficient)
- Encode AC refinement scans with correction-bit buffering
- Collect symbol frequency statistics for optimal Huffman table generation
- Flush pending EOBRUN symbols and restart interval markers

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMZERO`, `SIZEOF`, etc.)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_component_info`, `JHUFF_TBL`, `jpeg_destination_mgr`, `JBLOCKROW`, scan params (`Ss`, `Se`, `Ah`, `Al`)
- `jchuff.h` — `c_derived_tbl`, `jpeg_make_c_derived_tbl`, `jpeg_gen_optimal_table`
- **Defined elsewhere:** `jpeg_natural_order` (zigzag-to-natural scan order table), `jpeg_alloc_huff_table`, `JPEG_RST0`, `JERR_*` error codes, `ERREXIT`/`ERREXIT1` macros

# code/jpeg-6/jcprepct.c
## File Purpose
Implements the JPEG compression preprocessing controller, which manages the pipeline stage between raw input scanlines and the downsampler. It orchestrates color conversion, intermediate buffering, and vertical edge padding to satisfy the downsampler's row-group alignment requirements.

## Core Responsibilities
- Initialize and own the `my_prep_controller` object attached to `cinfo->prep`
- Accept raw input scanlines and drive the color converter (`cinfo->cconvert->color_convert`)
- Buffer color-converted rows until a full row group is ready for downsampling
- Invoke the downsampler (`cinfo->downsample->downsample`) on complete row groups
- Pad the bottom edge of the image by replicating the last real pixel row
- Pad downsampler output to a full iMCU height at image bottom
- Optionally support context-row mode (for input smoothing), providing wraparound row-pointer buffers

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, system headers)
- `jpeglib.h` / `jpegint.h` — `jpeg_compress_struct`, `jpeg_component_info`, `jpeg_c_prep_controller`, `JSAMPARRAY`, `JSAMPIMAGE`, `JDIMENSION`, `JPOOL_IMAGE`, `DCTSIZE`
- `jcopy_sample_rows` — defined elsewhere (likely `jutils.c`); copies rows within a sample array
- `cinfo->cconvert->color_convert` — color space converter, defined elsewhere
- `cinfo->downsample->downsample` — downsampling module, defined elsewhere
- `ERREXIT`, `MIN` — macros from JPEG error/utility headers

# code/jpeg-6/jcsample.c
## File Purpose
Implements the downsampling module for the IJG JPEG compressor. It reduces the spatial resolution of color components (chroma subsampling) from the input image resolution down to the component's coded resolution before DCT processing.

## Core Responsibilities
- Provide per-component downsampling method dispatch via `sep_downsample`
- Implement box-filter downsampling for arbitrary integer ratios (`int_downsample`)
- Implement optimized 1:1 passthrough (`fullsize_downsample`)
- Implement optimized 2h1v and 2h2v downsampling with alternating-bias dithering
- Implement smoothed variants of 2h2v and fullsize downsampling (conditional on `INPUT_SMOOTHING_SUPPORTED`)
- Handle horizontal edge padding via `expand_right_edge`
- Select and wire up the appropriate per-component method pointer during init

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMCOPY`, etc.)
- `jpeglib.h` / `jpegint.h` — `j_compress_ptr`, `jpeg_component_info`, `jpeg_downsampler`, `JSAMPARRAY`, `JDIMENSION`, `INT32`, `GETJSAMPLE`, `JMETHOD`, `DCTSIZE`, `MAX_COMPONENTS`
- `jcopy_sample_rows` — defined elsewhere (jutils.c); bulk row copy
- `ERREXIT`, `TRACEMS` — error/trace macros expanding to `cinfo->err` method calls

# code/jpeg-6/jctrans.c
## File Purpose
Implements JPEG transcoding compression: writing pre-existing raw DCT coefficient arrays directly to an output JPEG file, bypassing the normal pixel-data compression pipeline. Also provides utilities for copying critical image parameters from a decompression source to a compression destination.

## Core Responsibilities
- Initialize a compress object for coefficient-based (transcoding) output via `jpeg_write_coefficients`
- Copy lossless-transcoding-safe parameters from a decompressor to a compressor via `jpeg_copy_critical_parameters`
- Select and wire up the minimal set of compression modules needed for transcoding (`transencode_master_selection`)
- Implement a specialized coefficient buffer controller that reads from pre-supplied virtual arrays instead of a pixel pipeline
- Generate on-the-fly dummy DCT padding blocks at image right/bottom edges during output

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — all JPEG object types, method interfaces, constants
- **Defined elsewhere:** `jpeg_suppress_tables`, `jpeg_set_defaults`, `jpeg_set_colorspace`, `jpeg_alloc_quant_table`, `jinit_c_master_control`, `jinit_huff_encoder`, `jinit_phuff_encoder`, `jinit_marker_writer`, `jzero_far`

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

## External Dependencies
- **`jinclude.h`** — platform portability macros (`MEMZERO`, `SIZEOF`)
- **`jpeglib.h`** — all public JPEG types and struct definitions
- **`jpegint.h`** (via `JPEG_INTERNALS`) — internal module interfaces
- **`jerror.h`** (via `JPEG_INTERNALS`) — `ERREXIT1`, `WARNMS1`, `TRACEMS3` macros
- **Defined elsewhere:** `jinit_memory_mgr`, `jinit_marker_reader`, `jinit_input_controller` (module init functions); `jpeg_destroy`, `jpeg_abort` (`jcomapi.c`); `inputctl->consume_input`, `src->init_source`, `src->term_source`, `master->finish_output_pass` (subobject vtable methods)

# code/jpeg-6/jdapistd.c
## File Purpose
Implements the standard public API for the JPEG decompression pipeline, covering the full-decompression path from `jpeg_start_decompress` through scanline reading to buffered-image mode control. It is intentionally separated from `jdapimin.c` so that transcoder-only builds do not pull in the full decompressor.

## Core Responsibilities
- Initialize and drive the decompressor through its state machine (`DSTATE_*` transitions)
- Absorb multi-scan input into the coefficient buffer during startup
- Handle dummy output passes required by two-pass quantization
- Provide scanline-at-a-time output via `jpeg_read_scanlines`
- Provide raw iMCU-row output via `jpeg_read_raw_data`
- Manage buffered-image mode via `jpeg_start_output` / `jpeg_finish_output`

## External Dependencies
- `jinclude.h` — platform portability macros, system headers
- `jpeglib.h` — all public JPEG types and struct definitions; pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`
- `jinit_master_decompress` — defined in `jdmaster.c` (external)
- `cinfo->master->prepare_for_output_pass`, `finish_output_pass`, `is_dummy_pass` — implemented in `jdmaster.c`
- `cinfo->main->process_data` — implemented in `jdmainct.c`
- `cinfo->coef->decompress_data` — implemented in `jdcoefct.c`
- `cinfo->inputctl->consume_input`, `has_multiple_scans`, `eoi_reached` — implemented in `jdinput.c`
- `ERREXIT`, `ERREXIT1`, `WARNMS` — error macros resolving through `jerror.h` / `jdapimin.c`

# code/jpeg-6/jdatadst.c
## File Purpose
Implements a stdio-based JPEG compression data destination manager for the IJG JPEG library. It provides the output buffering and flushing logic that routes compressed JPEG bytes to a `FILE*` stream during encoding.

## Core Responsibilities
- Allocate and manage a 4096-byte output buffer for compressed JPEG data
- Flush the full buffer to disk via `fwrite` when it fills during compression
- Flush any remaining partial buffer bytes at end-of-compression
- Install the destination manager's three callback functions onto a `j_compress_ptr`
- Reuse an existing destination object if one is already attached to the compressor

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `JFWRITE` macro, `<stdio.h>`
- `jpeglib.h` — `j_compress_ptr`, `jpeg_destination_mgr`, `JOCTET`, `JPOOL_IMAGE`, `JPOOL_PERMANENT`, `boolean`
- `jerror.h` — `ERREXIT`, `JERR_FILE_WRITE`
- `fwrite`, `fflush`, `ferror` — C standard I/O (defined in `<stdio.h>`)
- `jpeg_start_compress`, `jpeg_finish_compress` — defined elsewhere; invoke the callbacks installed here

# code/jpeg-6/jdatasrc.c
## File Purpose
Implements a JPEG decompression data source manager that reads compressed JPEG data from an in-memory byte buffer (modified from the original stdio-based version). It satisfies the `jpeg_source_mgr` interface required by the IJG JPEG library.

## Core Responsibilities
- Provide a concrete `jpeg_source_mgr` implementation for memory-backed JPEG input
- Initialize and manage a fixed-size intermediate read buffer (`INPUT_BUF_SIZE = 4096`)
- Refill the decompressor's input buffer by copying from the in-memory source pointer
- Support skipping over unneeded data segments (APPn markers, etc.)
- Register all source manager callbacks on the `j_decompress_ptr` object via `jpeg_stdio_src`

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `memcpy` via `<string.h>`)
- `jpeglib.h` — `jpeg_source_mgr`, `j_decompress_ptr`, `JOCTET`, `JPOOL_PERMANENT`, `SIZEOF`, `jpeg_resync_to_restart`
- `jerror.h` — error/trace macros (included transitively; not directly used in this file)
- **Defined elsewhere:** `jpeg_resync_to_restart` (IJG library default restart-marker recovery)

# code/jpeg-6/jdcoefct.c
## File Purpose
Implements the coefficient buffer controller for JPEG decompression, sitting between entropy decoding and inverse-DCT stages. Manages both single-pass (streaming) and multi-pass (buffered-image, progressive) decompression modes, including optional interblock smoothing for progressive scans.

## Core Responsibilities
- Buffer MCU coefficient blocks received from the entropy decoder
- Drive the inverse-DCT (IDCT) transform per component block
- Coordinate input and output passes in multi-scan/buffered-image mode
- Implement JPEG K.8 interblock smoothing for progressive scans
- Initialize and wire up the `jpeg_d_coef_controller` vtable on startup

## External Dependencies
- `jinclude.h`, `jpeglib.h` (via `jpegint.h`, `jerror.h`)
- **Defined elsewhere:** `jzero_far`, `jcopy_block_row`, `jround_up`; all IDCT implementations (`inverse_DCT_method_ptr`); entropy decoder (`decode_mcu`); memory manager (`access_virt_barray`, `request_virt_barray`, `alloc_small`, `alloc_large`); input controller (`consume_input`, `finish_input_pass`)

# code/jpeg-6/jdcolor.c
## File Purpose
Implements output colorspace conversion for the IJG JPEG decompressor. It converts decoded JPEG component planes (YCbCr, YCCK, grayscale, CMYK) into the application's requested output colorspace (RGB, CMYK, grayscale, or pass-through).

## Core Responsibilities
- Build lookup tables for fixed-point YCbCr→RGB conversion coefficients
- Convert YCbCr component planes to interleaved RGB pixel rows
- Convert YCCK component planes to interleaved CMYK pixel rows
- Pass through grayscale (Y-only) data unchanged
- Perform null (same-colorspace) plane-to-interleaved reformatting
- Initialize and wire the `jpeg_color_deconverter` subobject into `cinfo`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMCOPY`
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_color_deconverter`, `JSAMPIMAGE`, `JSAMPARRAY`, `JDIMENSION`, `J_COLOR_SPACE` enum, `JPOOL_IMAGE`, `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE/PIXELSIZE`
- `jcopy_sample_rows` — defined in `jutils.c`
- `RIGHT_SHIFT`, `SHIFT_TEMPS`, `FIX` — macros from `jpegint.h`/`jmorecfg.h`
- `cinfo->mem->alloc_small` — memory manager defined in `jmemmgr.c`
- `cinfo->sample_range_limit` — populated by `jpeg_start_decompress` in `jdmaster.c`

# code/jpeg-6/jdct.h
## File Purpose
Private shared header for the JPEG DCT/IDCT subsystem within the Independent JPEG Group (IJG) library. It defines types, macros, and external declarations used by both the forward DCT (compression) and inverse DCT (decompression) modules and their per-algorithm implementation files.

## Core Responsibilities
- Define `DCTELEM` as the working integer type for forward DCT buffers (width depends on sample bit depth)
- Declare function pointer typedefs for forward DCT method dispatch (`forward_DCT_method_ptr`, `float_DCT_method_ptr`)
- Define per-algorithm multiplier types for IDCT dequantization tables (`ISLOW_MULT_TYPE`, `IFAST_MULT_TYPE`, `FLOAT_MULT_TYPE`)
- Provide the range-limiting macro `IDCT_range_limit` and `RANGE_MASK` for safe output clamping
- Declare all forward and inverse DCT routine entry points as `EXTERN`
- Supply fixed-point arithmetic macros (`FIX`, `DESCALE`, `MULTIPLY16C16`, `MULTIPLY16V16`) used across DCT implementation files
- Provide short-name aliases for linkers that cannot handle long external symbols

## External Dependencies
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JMETHOD`, `JPP`, `EXTERN`, `inverse_DCT_method_ptr`
- `jmorecfg.h` — `BITS_IN_JSAMPLE`, `MULTIPLIER`, `FAST_FLOAT`, `INT32`, `INT16`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- `RIGHT_SHIFT` — defined elsewhere (platform/compiler-specific macro, typically in `jconfig.h` or `jpegint.h`)
- `SHORTxSHORT_32`, `SHORTxLCONST_32` — optional compile-time flags defined by platform configuration; govern `MULTIPLY16C16` implementation

# code/jpeg-6/jddctmgr.c
## File Purpose
Manages the inverse DCT (IDCT) subsystem for JPEG decompression. It selects the appropriate IDCT implementation per component based on the requested DCT method and output scaling, and builds the corresponding dequantization multiplier tables used by the IDCT routines.

## Core Responsibilities
- Select the correct IDCT function pointer per component based on `DCT_scaled_size` and `dct_method`
- Convert raw zigzag-ordered quantization tables into method-specific multiplier tables (ISLOW, IFAST, FLOAT)
- Pre-zero multiplier tables so uninitialized components produce neutral gray output
- Cache the current IDCT method per component to avoid redundant table rebuilds
- Initialize and register the IDCT controller subobject with the decompressor

## External Dependencies
- `jinclude.h` — platform includes, `MEMZERO`, `SIZEOF`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JQUANT_TBL`, `J_DCT_METHOD` enum, `JPOOL_IMAGE`, `MAX_COMPONENTS`, `DCTSIZE`, `DCTSIZE2`
- `jdct.h` — `ISLOW_MULT_TYPE`, `IFAST_MULT_TYPE`, `FLOAT_MULT_TYPE`, `IFAST_SCALE_BITS`, `DESCALE`, `MULTIPLY16V16`, `SHIFT_TEMPS`, IDCT extern declarations
- **Defined elsewhere:** `jpeg_idct_islow`, `jpeg_idct_ifast`, `jpeg_idct_float`, `jpeg_idct_4x4`, `jpeg_idct_2x2`, `jpeg_idct_1x1` (individual IDCT implementation files); `jpeg_zigzag_order` (defined in `jutils.c`); `jpeg_inverse_dct` struct (defined in `jpegint.h`)

# code/jpeg-6/jdhuff.c
## File Purpose
Implements sequential (baseline) Huffman entropy decoding for the IJG JPEG library. It builds derived decoding tables from raw JHUFF_TBL data and decodes one MCU at a time from a compressed bitstream, supporting input suspension and restart marker handling.

## Core Responsibilities
- Build lookahead and min/max code tables from raw Huffman table data (`jpeg_make_d_derived_tbl`)
- Fill the bit-extraction buffer from the data source, handling FF/00 stuffing and end-of-data (`jpeg_fill_bit_buffer`)
- Decode a single Huffman symbol via slow-path bit-by-bit traversal when lookahead misses (`jpeg_huff_decode`)
- Decode one full MCU's DC and AC coefficients, writing dezigzagged output to `JBLOCKROW` (`decode_mcu`)
- Handle restart markers: flush bit buffer, re-read marker, reset DC predictors (`process_restart`)
- Initialize the entropy decoder module and wire up method pointers (`jinit_huff_decoder`)

## External Dependencies
- `jinclude.h` — platform includes, `MEMZERO`, `SIZEOF`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JHUFF_TBL`, `JBLOCKROW`, `JCOEF`
- `jdhuff.h` — `d_derived_tbl`, `bitread_*` types, `HUFF_DECODE`/`CHECK_BIT_BUFFER`/`GET_BITS` macros; shared with `jdphuff.c`
- `jpegint.h` (via `jpeglib.h` with `JPEG_INTERNALS`) — internal module structs
- `jerror.h` — `WARNMS`, `ERREXIT1`, warning/error codes
- **Defined elsewhere**: `jpeg_natural_order[]` (dezigzag table, used in `decode_mcu`); `cinfo->marker->read_restart_marker`; `cinfo->src->fill_input_buffer`

# code/jpeg-6/jdhuff.h
## File Purpose
Shared header for JPEG Huffman entropy decoding, providing derived table structures, bit-reading state types, and performance-critical inline macros used by both the sequential decoder (`jdhuff.c`) and progressive decoder (`jdphuff.c`).

## Core Responsibilities
- Define the `d_derived_tbl` structure for pre-computed Huffman lookup acceleration
- Define persistent and working bitreader state structures for MCU-boundary suspension support
- Provide `BITREAD_LOAD_STATE` / `BITREAD_SAVE_STATE` macros for register-level bit buffer management
- Expose `CHECK_BIT_BUFFER`, `GET_BITS`, `PEEK_BITS`, and `DROP_BITS` inline bit-extraction macros
- Expose the `HUFF_DECODE` macro implementing the fast lookahead decode path with slow fallback
- Declare the three out-of-line extern functions backing the macro fast paths

## External Dependencies
- `jpeglib.h` — `j_decompress_ptr`, `JHUFF_TBL`, `JOCTET`, `boolean`, `INT32`, `UINT8`, `JPP()`
- `jdhuff.c` — defines `jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`
- `jdphuff.c` — consumer of all three extern functions and all macros defined here

# code/jpeg-6/jdinput.c
## File Purpose
Implements the input controller module for the IJG JPEG decompressor. It orchestrates the state machine that alternates between reading JPEG markers (via `jdmarker.c`) and consuming compressed coefficient data (via the coefficient controller), dispatching to the appropriate submodule at each phase.

## Core Responsibilities
- Initialize the `jpeg_input_controller` subobject and wire up its method pointers
- Drive the marker-reading loop, detecting SOS and EOI markers
- Perform one-time image geometry setup on first SOS (`initial_setup`)
- Compute per-scan MCU layout for both interleaved and non-interleaved scans (`per_scan_setup`)
- Latch quantization tables at the start of each component's first scan (`latch_quant_tables`)
- Coordinate scan start/finish with the entropy decoder and coefficient controller
- Support full reset for re-use of a decompressor object

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` / `jpegint.h` — all JPEG types and submodule interfaces
- **Defined elsewhere:** `jdiv_round_up` (jutils.c), `cinfo->marker->read_markers` (jdmarker.c), `cinfo->entropy->start_pass` (jdhuff.c / jdphuff.c), `cinfo->coef->start_input_pass` / `consume_data` (jdcoefct.c), `cinfo->mem->alloc_small` (jmemmgr.c)

# code/jpeg-6/jdmainct.c
## File Purpose
Implements the main buffer controller for the JPEG decompressor, sitting between the coefficient decoder and the post-processor. It manages downsampled sample data in JPEG colorspace, optionally providing context rows (above/below neighbors) required by fancy upsampling algorithms.

## Core Responsibilities
- Allocate and manage the intermediate sample buffer between coefficient decode and post-processing
- Deliver iMCU row data to the post-processor as row groups
- Optionally maintain a "funny pointer" scheme to provide context rows without copying data
- Handle image top/bottom boundary conditions by duplicating edge sample rows
- Support a two-pass quantization crank mode that bypasses the main buffer entirely
- Initialize the `jpeg_d_main_controller` sub-object and wire it into `cinfo->main`

## External Dependencies
- **Includes:** `jinclude.h` (platform portability, `SIZEOF`, `MEMCOPY`), `jpeglib.h` (all JPEG types and sub-object interfaces, pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`)
- **Defined elsewhere:**
  - `jpeg_d_main_controller` (declared in `jpegint.h`)
  - `cinfo->coef->decompress_data` — coefficient controller
  - `cinfo->post->post_process_data` — post-processing controller
  - `cinfo->upsample->need_context_rows` — upsampler flag
  - `cinfo->mem->alloc_small`, `alloc_sarray` — memory manager
  - `ERREXIT`, `JPOOL_IMAGE`, `JBUF_PASS_THRU`, `JBUF_CRANK_DEST`, `METHODDEF`, `LOCAL`, `GLOBAL`, `JPP` — macros from `jpegint.h`/`jmorecfg.h`

# code/jpeg-6/jdmarker.c
## File Purpose
Implements JPEG datastream marker parsing for the IJG decompressor. It reads and decodes all standard JPEG markers (SOI, SOF, SOS, DHT, DQT, DRI, DAC, APP0, APP14, etc.) with full support for input suspension—if insufficient data is available, parsing aborts and resumes transparently on the next call.

## Core Responsibilities
- Scan the input stream for JPEG marker bytes (0xFF prefix sequences)
- Parse each marker's parameter segment and populate `j_decompress_ptr` fields
- Support suspendable I/O: return `FALSE` mid-parse if data runs out; resume on re-entry
- Install and dispatch per-marker handler function pointers (APPn, COM)
- Implement restart-marker synchronization and error recovery (`jpeg_resync_to_restart`)
- Initialize the `jpeg_marker_reader` subobject at decompressor creation time

## External Dependencies
- `jinclude.h` — system includes, `MEMCOPY`, `SIZEOF` macros
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_marker_reader`, `JHUFF_TBL`, `JQUANT_TBL`, `jpeg_component_info`, all `JPEG_*` status codes
- `jerror.h` — `ERREXIT`, `WARNMS2`, `TRACEMS*` macros
- **Defined elsewhere:** `jpeg_alloc_huff_table`, `jpeg_alloc_quant_table` (jcomapi.c); `datasrc->fill_input_buffer`, `skip_input_data`, `resync_to_restart` (source manager, e.g. jdatasrc.c)

# code/jpeg-6/jdmaster.c
## File Purpose
Master control module for the IJG JPEG decompressor. It selects which decompression sub-modules to activate, configures multi-pass quantization, and drives the per-pass setup/teardown lifecycle called by `jdapi.c`.

## Core Responsibilities
- Compute output image dimensions and DCT scaling factors (`jpeg_calc_output_dimensions`)
- Build the sample range-limit lookup table for fast pixel clamping (`prepare_range_limit_table`)
- Select and initialize all active decompressor sub-modules (IDCT, entropy decoder, upsampler, color converter, quantizer, buffer controllers)
- Manage per-output-pass start/finish sequencing and dummy-pass logic for 2-pass color quantization
- Expose `jinit_master_decompress` as the library entry point that boots the master object

## External Dependencies
- **Includes:** `jinclude.h`, `jpeglib.h` (pulls in `jpegint.h` and `jerror.h` via `JPEG_INTERNALS`)
- **Defined elsewhere:** `jdiv_round_up` (jutils.c), `jinit_1pass_quantizer` (jquant1.c), `jinit_2pass_quantizer` (jquant2.c), `jinit_merged_upsampler` (jdmerge.c), `jinit_color_deconverter` (jdcolor.c), `jinit_upsampler` (jdsample.c), `jinit_d_post_controller` (jdpostct.c), `jinit_inverse_dct` (jddctmgr.c), `jinit_phuff_decoder` (jdphuff.c), `jinit_huff_decoder` (jdhuff.c), `jinit_d_coef_controller` (jdcoefct.c), `jinit_d_main_controller` (jdmainct.c)

# code/jpeg-6/jdmerge.c
## File Purpose
Implements a merged upsampling and YCbCr-to-RGB color conversion pass for JPEG decompression. By combining chroma upsampling and colorspace conversion into a single loop, it avoids redundant per-pixel multiplications for the shared chroma terms, yielding a significant throughput improvement for the common 2h1v and 2h2v chroma subsampling cases.

## Core Responsibilities
- Build precomputed integer lookup tables for YCbCr→RGB channel contributions from Cb and Cr
- Provide a `start_pass` routine that resets per-pass state (spare row, row counter)
- Dispatch upsampling via `merged_2v_upsample` (2:1 vertical) or `merged_1v_upsample` (1:1 vertical)
- Implement `h2v1_merged_upsample`: process one luma row, 2:1 horizontal chroma replication, emit one output row
- Implement `h2v2_merged_upsample`: process two luma rows, 2:1 horizontal and 2:1 vertical chroma replication, emit two output rows
- Manage a spare row buffer for the 2v case when the caller supplies only a single-row output buffer, and discard the dummy last row on odd-height images
- Register itself as `cinfo->upsample` during module initialization

## External Dependencies
- `jinclude.h` — platform portability, `SIZEOF`, `MEMCOPY`
- `jpeglib.h` / `jpegint.h` — `j_decompress_ptr`, `jpeg_upsampler`, `JSAMPIMAGE`, `JDIMENSION`, `INT32`, `JSAMPLE`, `MAXJSAMPLE`, `CENTERJSAMPLE`, `GETJSAMPLE`, `RGB_RED/GREEN/BLUE`, `RGB_PIXELSIZE`, `JPOOL_IMAGE`, `SHIFT_TEMPS`, `RIGHT_SHIFT`, `FIX`
- `jcopy_sample_rows` — defined in `jutils.c` (used in spare-row drain path)
- `use_merged_upsample` — defined in `jdmaster.c` (controls whether this module is selected)
- `cinfo->sample_range_limit` — populated by `jdmaster.c` startup

# code/jpeg-6/jdphuff.c
## File Purpose
Implements progressive JPEG Huffman entropy decoding for the IJG JPEG library. It handles all four scan types defined by the progressive JPEG standard: DC initial, DC refinement, AC initial, and AC refinement scans, with full support for input suspension (backtracking to MCU start on partial data).

## Core Responsibilities
- Initialize the progressive Huffman decoder state per scan pass (`start_pass_phuff_decoder`)
- Validate progressive scan parameters (Ss, Se, Ah, Al) and update coefficient progression status
- Decode DC coefficients for initial scans with delta-coding and bit-shifting
- Decode AC coefficients for initial scans including EOB run-length handling
- Decode DC/AC refinement scans (successive approximation bit-plane refinement)
- Handle restart markers and resynchronize decoder state
- Allocate and initialize the `phuff_entropy_decoder` object

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JBLOCKROW`, `JCOEF`, scan parameter fields
- `jdhuff.h` — `d_derived_tbl`, `bitread_perm_state`, `bitread_working_state`, `HUFF_DECODE`, `CHECK_BIT_BUFFER`, `GET_BITS`, `BITREAD_*` macros
- **Defined elsewhere:** `jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`, `jpeg_natural_order`, `ERREXIT*`/`WARNMS*` error macros

# code/jpeg-6/jdpostct.c
## File Purpose
Implements the JPEG decompression postprocessing controller, which manages the pipeline stage between upsampling/color-conversion and color quantization/reduction. It buffers decoded pixel data in either a single strip or a full-image virtual array depending on the quantization pass mode.

## Core Responsibilities
- Initialize and own the strip buffer or full-image virtual array used between upsample and quantize stages
- Select the correct processing function pointer (`post_process_data`) based on the current pass mode
- Drive the upsample→quantize pipeline for one-pass color quantization
- Buffer full-image rows during the first pass of two-pass color quantization (prepass, no output emitted)
- Re-read buffered rows and quantize+emit them during the second pass of two-pass quantization
- Short-circuit the postprocessing stage entirely when no color quantization is needed (delegate directly to upsampler)

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — `jpeg_decompress_struct`, `jpeg_d_post_controller`, `jvirt_sarray_ptr`, `JSAMPIMAGE`, `JSAMPARRAY`, `JDIMENSION`, `J_BUF_MODE`, `JPOOL_IMAGE`
- **Defined elsewhere:** `jround_up` (math utility), `ERREXIT` (error macro), `cinfo->upsample->upsample`, `cinfo->cquantize->color_quantize`, `cinfo->mem->*` (memory manager vtable)

# code/jpeg-6/jdsample.c
## File Purpose
Implements the upsampling stage of the JPEG decompression pipeline. It expands chroma (and other subsampled) components back to full output resolution, optionally using bilinear ("fancy") interpolation or simple box-filter replication.

## Core Responsibilities
- Initialize the upsampler module and select per-component upsample methods during decompression setup
- Buffer one row group of upsampled data in `color_buf` before passing to color conversion
- Support multiple upsampling strategies: fullsize passthrough, no-op, integer box-filter, fast 2h1v/2h2v box, and fancy triangle-filter variants
- Track remaining image rows to handle images whose height is not a multiple of `max_v_samp_factor`
- Allocate intermediate color conversion buffers only for components that actually require rescaling

## External Dependencies
- `jinclude.h` — platform portability macros
- `jpeglib.h` / `jpegint.h` — JPEG decompressor structs, `jpeg_upsampler`, `jpeg_component_info`, sample typedefs
- `jcopy_sample_rows` — defined in `jutils.c`
- `jround_up` — defined in `jutils.c`
- `ERREXIT` — error macro from `jerror.h`

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

## External Dependencies
- **`jinclude.h`** — platform portability macros (`SIZEOF`, `MEMCOPY`, system headers)
- **`jpeglib.h`** — all public JPEG types and state machine structs; includes `jpegint.h` and `jerror.h` when `JPEG_INTERNALS` is defined
- **Defined elsewhere:**
  - `jinit_huff_decoder` — sequential Huffman decoder init (`jdhuff.c`)
  - `jinit_phuff_decoder` — progressive Huffman decoder init (`jdphuff.c`)
  - `jinit_d_coef_controller` — coefficient buffer controller init (`jdcoefct.c`)
  - `ERREXIT`, `ERREXIT1` — error macros resolving via `cinfo->err->error_exit`
  - `DSTATE_READY`, `DSTATE_RDCOEFS`, `DSTATE_STOPPING` — decompressor state constants (`jpegint.h`)

# code/jpeg-6/jerror.c
## File Purpose
This is a Quake III Arena-adapted version of the IJG JPEG library's error-handling module. It replaces the standard Unix `stderr`-based error output with Quake's renderer interface (`ri.Error` and `ri.Printf`), integrating JPEG decode/encode errors into the engine's error and logging systems.

## Core Responsibilities
- Define and populate the JPEG standard message string table from `jerror.h`
- Implement the `error_exit` handler that calls `ri.Error(ERR_FATAL, ...)` on fatal JPEG errors
- Implement `output_message` to route JPEG messages through `ri.Printf`
- Implement `emit_message` with warning-level filtering and trace-level gating
- Implement `format_message` to produce formatted error strings from message codes and parameters
- Implement `reset_error_mgr` to clear error state between images
- Provide `jpeg_std_error` to wire all handler function pointers into a `jpeg_error_mgr`

## External Dependencies
- `jinclude.h` — platform-specific includes and memory macros
- `jpeglib.h` — JPEG library types and struct definitions
- `jversion.h` — version string constants embedded in the message table
- `jerror.h` — message code enum and `JMESSAGE` macro (included twice via X-macro pattern)
- `../renderer/tr_local.h` — provides `ri` (`refimport_t`) for `ri.Error` and `ri.Printf`; **defined elsewhere** in the renderer module
- `jpeg_destroy` — defined elsewhere in the IJG library (`jcomapi.c`)
- `ri.Error`, `ri.Printf` — defined elsewhere; renderer import table populated at renderer initialization

# code/jpeg-6/jerror.h
## File Purpose
Defines all error and trace message codes for the IJG JPEG library as a `J_MESSAGE_CODE` enum, and provides a set of convenience macros for emitting fatal errors, warnings, and trace/debug messages through the JPEG library's error manager vtable.

## Core Responsibilities
- Declares the `J_MESSAGE_CODE` enum by expanding `JMESSAGE` macros into enum values
- Provides `ERREXIT`/`ERREXIT1–4`/`ERREXITS` macros for fatal error dispatch (calls `error_exit` function pointer)
- Provides `WARNMS`/`WARNMS1–2` macros for non-fatal/corrupt-data warnings (calls `emit_message` at level -1)
- Provides `TRACEMS`/`TRACEMS1–8`/`TRACEMSS` macros for informational and debug tracing (calls `emit_message` at caller-supplied level)
- Supports dual-inclusion pattern: first include builds the enum, second include (with `JMESSAGE` defined externally) builds a string table

## External Dependencies
- No `#include` directives in this file.
- `JCOPYRIGHT`, `JVERSION`: string macros, defined in `jversion.h` (included elsewhere).
- `JMSG_STR_PARM_MAX`: integer constant, defined in `jpeglib.h`.
- `j_common_ptr`, `j_compress_ptr`, `j_decompress_ptr`: typedefs defined in `jpeglib.h`.
- `strncpy`: standard C library, used in `ERREXITS` and `TRACEMSS`.
- `error_exit`, `emit_message`: function pointer fields on `jpeg_error_mgr`, defined/populated in `jerror.c`.

# code/jpeg-6/jfdctflt.c
## File Purpose
Implements the forward Discrete Cosine Transform (DCT) using floating-point arithmetic for the IJG JPEG library. It applies the Arai, Agui, and Nakajima (AA&N) scaled DCT algorithm to an 8×8 block of samples in-place, performing two separable 1-D passes (rows then columns).

## Core Responsibilities
- Accept a flat 64-element `FAST_FLOAT` array representing an 8×8 sample block
- Apply 1-D forward DCT across all 8 rows (Pass 1)
- Apply 1-D forward DCT down all 8 columns (Pass 2)
- Produce scaled DCT coefficients in-place (scaling deferred to quantization step)
- Guard entire implementation under `#ifdef DCT_FLOAT_SUPPORTED`

## External Dependencies
- `jinclude.h` — system include abstraction, `MEMZERO`/`MEMCOPY` macros
- `jpeglib.h` — JPEG library types (`FAST_FLOAT`, `DCTSIZE`, `GLOBAL`, etc.)
- `jdct.h` — DCT subsystem private declarations; declares `jpeg_fdct_float` extern and `float_DCT_method_ptr`
- `FAST_FLOAT` — defined elsewhere (in `jmorecfg.h` via `jpeglib.h`)
- `DCTSIZE` — defined as `8` in `jpeglib.h`
- `DCT_FLOAT_SUPPORTED` — compile-time feature flag, defined elsewhere (typically `jconfig.h`)

# code/jpeg-6/jfdctfst.c
## File Purpose
Implements the fast, lower-accuracy integer forward Discrete Cosine Transform (DCT) for the IJG JPEG library. It applies the Arai, Agui & Nakajima (AA&N) scaled DCT algorithm to an 8×8 block of DCT elements in-place, using only 5 multiplies and 29 adds per 1-D pass.

## Core Responsibilities
- Perform a 2-pass (row then column) separable 8-point 1-D forward DCT on a single 8×8 block
- Encode fixed-point multiplicative constants at 8 fractional bits (`CONST_BITS = 8`)
- Provide an optionally less-accurate descale path (`USE_ACCURATE_ROUNDING` not defined → plain right-shift)
- Guard the entire implementation behind `#ifdef DCT_IFAST_SUPPORTED`
- Write scaled DCT coefficients back into the input buffer in-place (output×8 convention per JPEG spec)

## External Dependencies
- `jinclude.h` — platform portability, `MEMZERO`/`MEMCOPY`, system headers
- `jpeglib.h` — top-level JPEG library types and constants (`DCTSIZE`, `DCTSIZE2`)
- `jdct.h` — `DCTELEM` typedef, `DESCALE`, `RIGHT_SHIFT`, `FIX`, `SHIFT_TEMPS`, forward DCT extern declarations
- `DCTELEM`, `INT32`, `RIGHT_SHIFT`, `SHIFT_TEMPS` — defined elsewhere (`jmorecfg.h`, `jdct.h`, compiler/platform headers)
- `DCT_IFAST_SUPPORTED` — configuration macro defined in `jconfig.h`

# code/jpeg-6/jfdctint.c
## File Purpose
Implements the slow-but-accurate integer forward Discrete Cosine Transform (FDCT) for the IJG JPEG library. It performs a separable 2-D 8×8 DCT using a scaled fixed-point version of the Loeffler-Ligtenberg-Moschytz algorithm with 12 multiplies and 32 adds per 1-D pass.

## Core Responsibilities
- Perform the forward DCT on a single 8×8 block of `DCTELEM` samples in-place
- Execute two separable 1-D DCT passes: first across all 8 rows, then all 8 columns
- Apply scaled fixed-point integer arithmetic to avoid floating-point at runtime
- Scale outputs by `sqrt(8) * 2^PASS1_BITS` after pass 1; remove `PASS1_BITS` scaling after pass 2, leaving a net factor-of-8 scale (consumed by the quantization step in `jcdctmgr.c`)

## External Dependencies
- `jinclude.h` — system include abstraction, `MEMZERO`/`MEMCOPY`, `<stdio.h>`, `<string.h>`
- `jpeglib.h` — `DCTSIZE`, `INT32`, `BITS_IN_JSAMPLE`, `JSAMPLE`, JPEG object types
- `jdct.h` — `DCTELEM`, `DESCALE`, `FIX`, `MULTIPLY16C16`, `CONST_SCALE`, `ONE`; declares `jpeg_fdct_islow` as `EXTERN`
- `SHIFT_TEMPS`, `RIGHT_SHIFT` — defined elsewhere (platform-specific, typically in `jmorecfg.h` or `jpegint.h`)
- `MULTIPLY16C16` — defined in `jdct.h`, platform-tunable for 16×16→32 multiply optimization
- `DCT_ISLOW_SUPPORTED` — compile-time feature flag, defined elsewhere in the build configuration

# code/jpeg-6/jidctflt.c
## File Purpose
Implements a floating-point inverse DCT (IDCT) with integrated dequantization for the IJG JPEG library. It converts an 8×8 block of quantized DCT coefficients back into pixel-domain sample values using the Arai, Agui, and Nakajima (AA&N) scaled DCT algorithm.

## Core Responsibilities
- Dequantize input JCOEF coefficients by multiplying against the component's float multiplier table
- Perform a separable 2-pass (column then row) 8-point floating-point IDCT
- Short-circuit columns where all AC terms are zero (DC-only optimization)
- Descale results by factor of 8 (2³) in the row pass
- Range-limit final values to valid `JSAMPLE` range via lookup table
- Write output samples into the caller-supplied output row buffer

## External Dependencies
- `jinclude.h` — platform includes (`stdio.h`, `string.h`, etc.) and utility macros
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `FLOAT_MULT_TYPE`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `FAST_FLOAT`
- `jmorecfg.h` (via `jpeglib.h`) — `FAST_FLOAT`, `MULTIPLIER`, `CENTERJSAMPLE`, `MAXJSAMPLE`, `SHIFT_TEMPS`
- `compptr->dct_table` — populated externally by `jddctmgr.c` during decompressor startup
- `cinfo->sample_range_limit` — populated externally by `jdmaster.c` (`prepare_range_limit_table`)

# code/jpeg-6/jidctfst.c
## File Purpose
Implements a fast, reduced-accuracy integer Inverse Discrete Cosine Transform (IDCT) for the IJG JPEG decompression library. It performs simultaneous dequantization and 8x8 block IDCT using the Arai, Agui & Nakajima (AA&N) scaled algorithm, trading precision for speed compared to the slow/accurate variant (`jidctint.c`).

## Core Responsibilities
- Dequantize 64 DCT coefficients using a precomputed multiplier table (`compptr->dct_table`)
- Execute a two-pass separable 1-D IDCT (columns first, then rows) on a single 8x8 block
- Short-circuit computation for columns/rows with all-zero AC terms (DC-only fast path)
- Scale and range-limit all 64 output pixels into valid `JSAMPLE` values (0–MAXJSAMPLE)
- Write one reconstructed 8x8 tile of pixel data into the caller-supplied output buffer

## External Dependencies
- `jinclude.h` — platform portability, `MEMZERO`/`MEMCOPY`, system headers
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `DCTELEM`, `IFAST_MULT_TYPE`, `IFAST_SCALE_BITS`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `FIX`
- `jmorecfg.h` (via jpeglib.h) — `BITS_IN_JSAMPLE`, `MULTIPLIER`, `INT32`, `MAXJSAMPLE`, `CENTERJSAMPLE`
- **Defined elsewhere:** `IDCT_range_limit` result table populated by `jdmaster.c:prepare_range_limit_table`; `compptr->dct_table` populated by `jddctmgr.c`; `DCT_IFAST_SUPPORTED` guard defined in `jconfig.h`

# code/jpeg-6/jidctint.c
## File Purpose
Implements the slow-but-accurate integer inverse DCT (IDCT) for the IJG JPEG library, performing combined dequantization and 2D IDCT on a single 8×8 DCT coefficient block. This is the `JDCT_ISLOW` variant, based on the Loeffler–Ligtenberg–Moschytz algorithm with 12 multiplies and 32 adds per 1-D pass.

## Core Responsibilities
- Dequantize 64 DCT coefficients using the component's quantization multiplier table
- Execute a two-pass separable 2D IDCT (column pass then row pass) on an 8×8 block
- Apply scaled fixed-point arithmetic with compile-time integer constants to avoid floating-point at runtime
- Short-circuit all-zero AC columns/rows for a common-case speedup
- Range-limit and clamp all 64 output samples into valid `JSAMPLE` (0–255) values
- Write the decoded 8×8 pixel block into the caller-supplied output scanline buffer

## External Dependencies

- `jinclude.h` — platform includes, `MEMZERO`/`MEMCOPY`, `size_t`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `ISLOW_MULT_TYPE`, `DESCALE`, `FIX`, `MULTIPLY16C16`, `IDCT_range_limit`, `RANGE_MASK`, `CONST_SCALE`
- `jmorecfg.h` (via `jpeglib.h`) — `INT32`, `MULTIPLIER`, `BITS_IN_JSAMPLE`, `CENTERJSAMPLE`, `MAXJSAMPLE`, `SHIFT_TEMPS`, `RIGHT_SHIFT`
- **Defined elsewhere:** `IDCT_range_limit` table populated by `jdmaster.c:prepare_range_limit_table()`; multiplier table (`compptr->dct_table`) populated by `jddctmgr.c`

# code/jpeg-6/jidctred.c
## File Purpose
Implements reduced-size inverse DCT (IDCT) routines for JPEG decompression, producing 4x4, 2x2, or 1x1 pixel output from an 8x8 DCT coefficient block. These are used when downscaled image output is requested, avoiding a full 8x8 IDCT followed by downsampling.

## Core Responsibilities
- Dequantize DCT coefficients using the component's quantization table
- Perform a two-pass (column then row) reduced IDCT using simplified LL&M butterfly arithmetic
- Clamp output samples to valid range via a pre-built range-limit lookup table
- Write reduced-size pixel rows directly into the output sample buffer
- Short-circuit all-zero AC coefficient cases for performance

## External Dependencies

- `jinclude.h` — platform portability macros (`MEMZERO`, etc.)
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JCOEFPTR`, `JSAMPARRAY`, `JDIMENSION`, `JSAMPLE`, `JSAMPROW`
- `jdct.h` — `ISLOW_MULT_TYPE`, `IDCT_range_limit`, `RANGE_MASK`, `DESCALE`, `FIX`, `MULTIPLY16C16`, DCT size/precision constants
- `DCTSIZE`, `BITS_IN_JSAMPLE`, `INT32`, `MAXJSAMPLE`, `CENTERJSAMPLE` — defined elsewhere in the JPEG library configuration headers
- `RIGHT_SHIFT`, `SHIFT_TEMPS` — platform-specific shift helpers defined elsewhere

# code/jpeg-6/jinclude.h
## File Purpose
A portability header for the Independent JPEG Group (IJG) JPEG library that centralizes system include file selection and provides cross-platform abstraction macros. It resolves platform differences in standard library availability, string function families, and I/O operations so the rest of the JPEG library can use a uniform interface.

## Core Responsibilities
- Suppresses MSVC compiler warnings when building on Win32 targets
- Conditionally includes system headers (`stddef.h`, `stdlib.h`, `sys/types.h`, `stdio.h`) based on `jconfig.h` feature flags
- Abstracts BSD vs. ANSI/SysV string/memory functions (`bzero`/`bcopy` vs. `memset`/`memcpy`) behind `MEMZERO`/`MEMCOPY` macros
- Provides a `SIZEOF()` macro to guarantee `size_t` return from `sizeof()` on non-conforming compilers
- Provides `JFREAD`/`JFWRITE` macros wrapping `fread`/`fwrite` with portable argument casting

## External Dependencies
- `../jpeg-6/jconfig.h` — Watcom-targeted auto-configuration header; defines `HAVE_STDDEF_H`, `HAVE_STDLIB_H`, `NEED_BSD_STRINGS` absence, `NEED_SYS_TYPES_H` absence, and DCT defaults
- `<stddef.h>`, `<stdlib.h>`, `<sys/types.h>` — conditionally included per `jconfig.h` flags
- `<stdio.h>` — unconditionally included (required for `FILE` references in `jpeglib.h`)
- `<string.h>` or `<strings.h>` — selected at compile time for memory/string primitives
- `fread`, `fwrite`, `memset`/`memcpy` or `bzero`/`bcopy` — defined in the C standard library, not in this file

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

## External Dependencies
- `../game/q_shared.h` — `fileHandle_t`, `qboolean`, `Z_Malloc`
- `../qcommon/qcommon.h` — `FS_FOpenFileRead`, `FS_FCloseFile`
- `jpeglib.h` (local jpeg-6) — entire libjpeg decompression API
- **Defined elsewhere:** `FS_FOpenFileRead`, `FS_FCloseFile` (filesystem module); `Z_Malloc` (zone memory, `qcommon/`); all `jpeg_*` functions (libjpeg-6 implementation files in `code/jpeg-6/`)

# code/jpeg-6/jmemansi.c
## File Purpose
Provides the ANSI-standard, system-dependent memory management backend for the IJG JPEG library. It implements allocation via standard `malloc`/`free` and temporary file backing store via POSIX `tmpfile()` for overflow when available memory is insufficient.

## Core Responsibilities
- Allocate and free "small" heap objects via `malloc`/`free`
- Allocate and free "large" heap objects (same mechanism on flat-memory systems)
- Report available memory to the JPEG memory manager
- Create, read, write, and close temporary backing-store files using `tmpfile()`
- Provide memory subsystem init/term hooks (trivial in this implementation)

## External Dependencies
- `jinclude.h` — platform includes, `JFREAD`/`JFWRITE` macros, `SIZEOF`
- `jpeglib.h` — `j_common_ptr`, `jpeg_common_struct`, `jpeg_memory_mgr`
- `jmemsys.h` — `backing_store_info`, `backing_store_ptr`, function prototypes
- `malloc`, `free` — C standard library heap (ANSI `<stdlib.h>`)
- `tmpfile`, `fseek`, `fclose` — C standard I/O (`<stdio.h>`)
- `ERREXIT`, `ERREXITS` — defined elsewhere in the JPEG library (`jerror.h` / `jmemmgr.c`); perform error exit via `cinfo->err->error_exit`

# code/jpeg-6/jmemdos.c
## File Purpose
MS-DOS-specific implementation of the IJG JPEG memory manager's system-dependent layer. Provides heap allocation and three types of backing store (DOS files, XMS extended memory, EMS expanded memory) for spilling JPEG working buffers when RAM is insufficient.

## Core Responsibilities
- Allocate and free small (near heap) and large (far heap) memory blocks
- Report available memory to the JPEG memory manager
- Select and generate unique temporary file names using the `TMP`/`TEMP` environment variables
- Open, read, write, and close DOS-file-based backing store via direct DOS calls (assembly stubs)
- Open, read, write, and close XMS (extended memory, V2.0) backing store via the XMS driver
- Open, read, write, and close EMS (expanded memory, LIM/EMS 4.0) backing store via the EMS driver
- Initialize and terminate the memory subsystem (`jpeg_mem_init`, `jpeg_mem_term`)

## External Dependencies
- `jinclude.h`, `jpeglib.h`, `jmemsys.h` — IJG JPEG library internals
- `<alloc.h>` (Turbo C) or `<malloc.h>` (MSVC) — far-heap routines
- `<stdlib.h>` — `malloc`, `free`, `getenv`
- Assembly stubs in `jmemdosa.asm` (defined elsewhere): `jdos_open`, `jdos_close`, `jdos_seek`, `jdos_read`, `jdos_write`, `jxms_getdriver`, `jxms_calldriver`, `jems_available`, `jems_calldriver`
- `ERREXIT`, `ERREXITS`, `TRACEMSS`, `TRACEMS1` — error/trace macros defined in `jerror.h` (via `jpeglib.h`)

# code/jpeg-6/jmemmgr.c
## File Purpose
Implements the system-independent JPEG memory manager for the IJG JPEG library. It provides pool-based allocation (small and large objects), 2-D array allocation for image samples and DCT coefficient blocks, and virtual array management with optional disk-backed overflow storage.

## Core Responsibilities
- Pool-based allocation and lifetime management of "small" and "large" memory objects across `JPOOL_PERMANENT` and `JPOOL_IMAGE` lifetimes
- Allocation of 2-D sample arrays (`JSAMPARRAY`) and coefficient-block arrays (`JBLOCKARRAY`) with chunked large-object backing
- Registration and deferred realization of virtual (potentially disk-backed) sample and block arrays
- Swapping virtual array strips between in-memory buffers and backing store on demand
- Tracking total allocated space and enforcing `max_memory_to_use` policy
- Teardown: freeing all pools (including closing backing-store files) and destroying the manager itself

## External Dependencies
- `jinclude.h`, `jpeglib.h`, `jmemsys.h`
- **Defined elsewhere:** `jpeg_get_small`, `jpeg_free_small`, `jpeg_get_large`, `jpeg_free_large`, `jpeg_mem_available`, `jpeg_open_backing_store`, `jpeg_mem_init`, `jpeg_mem_term` (system-dependent, e.g., `jmemnobs.c` or `jmemansi.c`), `jzero_far`, `ERREXIT`/`ERREXIT1` macros (error handler)

# code/jpeg-6/jmemname.c
## File Purpose
Implements the system-dependent portion of the IJG JPEG memory manager for systems that require explicit temporary file naming. It provides memory allocation, memory availability reporting, and backing-store (temp file) management using named temporary files on disk.

## Core Responsibilities
- Allocate and free small and large memory objects via `malloc`/`free`
- Report available memory to the JPEG memory manager
- Generate unique temporary file names (via `mktemp` or manual polling)
- Open, read, write, and close backing-store temp files
- Initialize and terminate the memory subsystem

## External Dependencies
- `jinclude.h` — platform stdio/string includes, `JFREAD`/`JFWRITE` macros
- `jpeglib.h` — `j_common_ptr`, `jpeg_memory_mgr`, `ERREXIT`, `TRACEMSS`
- `jmemsys.h` — `backing_store_ptr`, `backing_store_info`, function signatures
- `<errno.h>` — `ENOENT` (conditional, `NO_MKTEMP` path only)
- `malloc`, `free` — defined in `<stdlib.h>` or declared extern
- `mktemp`, `unlink`, `fopen`, `fclose`, `fseek` — defined in system libc

# code/jpeg-6/jmemnobs.c
## File Purpose
Provides the Quake III renderer-integrated system-dependent JPEG memory manager backend. It replaces standard `malloc`/`free` with the renderer's `ri.Malloc`/`ri.Free` allocator functions, ensuring JPEG memory operations go through the engine's tracked heap. Backing store (disk temp files) is explicitly unsupported.

## Core Responsibilities
- Implement `jpeg_get_small`/`jpeg_free_small` via `ri.Malloc`/`ri.Free`
- Implement `jpeg_get_large`/`jpeg_free_large` via the same allocator (no distinction between small/large)
- Report unlimited available memory to the JPEG library (`jpeg_mem_available`)
- Unconditionally error out if backing store is ever requested (`jpeg_open_backing_store`)
- Provide no-op init/term lifecycle stubs (`jpeg_mem_init`/`jpeg_mem_term`)

## External Dependencies
- `jinclude.h` — platform include shims, `SIZEOF`, `MEMCOPY`, etc.
- `jpeglib.h` — JPEG library types (`j_common_ptr`, `backing_store_ptr`, `ERREXIT`, `JERR_NO_BACKING_STORE`)
- `jmemsys.h` — declares the function signatures this file implements
- `../renderer/tr_local.h` — exposes `extern refimport_t ri`, providing `ri.Malloc` and `ri.Free`
- `ri` (`refimport_t`) — defined elsewhere in the renderer; this file depends on it being initialized before any JPEG operation occurs

# code/jpeg-6/jmemsys.h
## File Purpose
Defines the interface between the system-independent JPEG memory manager (`jmemmgr.c`) and its system-dependent backend implementations. It declares the contract that any platform-specific memory manager must fulfill, covering small/large heap allocation, available-memory querying, and backing-store (temp file/XMS/EMS) management.

## Core Responsibilities
- Declare small-heap allocation/free functions (`jpeg_get_small`, `jpeg_free_small`)
- Declare large-heap allocation/free functions (`jpeg_get_large`, `jpeg_free_large`)
- Declare available-memory query (`jpeg_mem_available`)
- Define the `backing_store_info` struct with vtable-style method pointers for temp-file I/O
- Declare backing-store lifecycle functions (`jpeg_open_backing_store`, `jpeg_mem_init`, `jpeg_mem_term`)
- Provide short-name aliases for linkers with limited symbol-length support (`NEED_SHORT_EXTERNAL_NAMES`)

## External Dependencies
- `jpeglib.h` / `jpegint.h` — for `j_common_ptr`, `JMETHOD`, `JPP`, `FAR`, `EXTERN` macros
- `jconfig.h` — supplies `USE_MSDOS_MEMMGR`, `NEED_SHORT_EXTERNAL_NAMES`, `MAX_ALLOC_CHUNK` overrides
- `<stdio.h>` — `FILE *` used in the non-DOS `backing_store_info` branch
- All declared functions are **defined elsewhere** in one of: `jmemansi.c`, `jmemnobs.c`, `jmemdos.c`, `jmemname.c` (selected at build time)

# code/jpeg-6/jmorecfg.h
## File Purpose
Platform-portability and capability configuration header for the Independent JPEG Group (IJG) JPEG library. It defines primitive typedefs, compile-time capability switches, and machine-dependent tuning macros used throughout the JPEG codec.

## Core Responsibilities
- Define `JSAMPLE` (pixel sample type) and `JCOEF` (DCT coefficient type) based on bit-depth setting
- Provide portable integer typedefs (`UINT8`, `UINT16`, `INT16`, `INT32`, `JDIMENSION`, `JOCTET`)
- Guard against `unsigned char` / `char` signedness portability issues via `GETJSAMPLE`/`GETJOCTET` macros
- Declare function-linkage macros (`METHODDEF`, `LOCAL`, `GLOBAL`, `EXTERN`)
- Enable/disable encoder and decoder feature modules at compile time
- Configure RGB scanline channel ordering and pixel stride
- Provide performance hints: `INLINE`, `MULTIPLIER`, `FAST_FLOAT`

## External Dependencies
- No includes of its own.
- Consumed by: `jpeglib.h`, and transitively all `j*.c` translation units in `code/jpeg-6/`.
- Conditioned on external macros: `HAVE_UNSIGNED_CHAR`, `HAVE_UNSIGNED_SHORT`, `CHAR_IS_UNSIGNED`, `NEED_FAR_POINTERS`, `XMD_H`, `HAVE_PROTOTYPES`, `HAVE_BOOLEAN`, `JPEG_INTERNALS`, `__GNUC__` — all expected to be set (or absent) by `jconfig.h` or the build system.

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

## External Dependencies
- `jpeglib.h` — defines `j_compress_ptr`, `j_decompress_ptr`, `JSAMPARRAY`, `JBLOCKROW`, `JDIMENSION`, `jvirt_barray_ptr`, `jpeg_component_info`, `jpeg_marker_parser_method`, `JMETHOD`, `JPP`, `EXTERN`
- `jmorecfg.h` (transitively) — `INT32`, `boolean`, `FAR`, `MAX_COMPONENTS`
- Symbols defined elsewhere: all `jinit_*` bodies live in their respective `.c` modules (`jcmaster.c`, `jdmaster.c`, `jdmarker.c`, etc.); utility bodies in `jutils.c`

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

## External Dependencies

- `code/jpeg-6/jconfig.h` — Platform/compiler configuration flags (`HAVE_PROTOTYPES`, `HAVE_UNSIGNED_CHAR`, `JDCT_DEFAULT`, etc.)
- `code/jpeg-6/jmorecfg.h` — Type definitions (`JSAMPLE`, `JCOEF`, `JOCTET`, `UINT8`, `UINT16`, `INT32`, `JDIMENSION`), linkage macros (`EXTERN`, `METHODDEF`, `LOCAL`)
- `code/jpeg-6/jpegint.h` — Internal submodule struct definitions (included only when `JPEG_INTERNALS` is defined)
- `code/jpeg-6/jerror.h` — Error/message code enum and `ERREXIT`/`WARNMS`/`TRACEMS` macros (included only when `JPEG_INTERNALS` is defined)
- All internal submodule structs (`jpeg_comp_master`, `jpeg_entropy_encoder`, `jpeg_inverse_dct`, etc.) are defined elsewhere (in `jpegint.h`) and referenced here only as forward-declared pointers

# code/jpeg-6/jpegtran.c
## File Purpose
A standalone command-line application for lossless JPEG transcoding. It reads a JPEG file as raw DCT coefficients and rewrites it with different encoding parameters (progressive, arithmetic coding, restart intervals, etc.) without a full decode/re-encode cycle.

## Core Responsibilities
- Parse command-line switches to configure a JPEG compression context
- Open input/output files (or fall back to stdin/stdout)
- Decompress source JPEG into DCT coefficient arrays (lossless read)
- Copy critical parameters from source to destination compressor
- Re-compress using DCT arrays directly, preserving image quality
- Clean up all JPEG objects and file handles on exit

## External Dependencies

- **`cdjpeg.h`** — IJG common application declarations; provides `keymatch`, `read_stdin`, `write_stdout`, `read_scan_script`, `start_progress_monitor`, `end_progress_monitor`, `enable_signal_catcher`, `READ_BINARY`, `WRITE_BINARY`, `TWO_FILE_COMMANDLINE`
- **`jversion.h`** — `JVERSION`, `JCOPYRIGHT` string macros
- **Defined elsewhere (IJG library):** `jpeg_create_decompress`, `jpeg_create_compress`, `jpeg_std_error`, `jpeg_read_header`, `jpeg_read_coefficients`, `jpeg_copy_critical_parameters`, `jpeg_write_coefficients`, `jpeg_finish_compress`, `jpeg_destroy_compress`, `jpeg_finish_decompress`, `jpeg_destroy_decompress`, `jpeg_stdio_src`, `jpeg_stdio_dest`, `jpeg_simple_progression`, `j_compress_ptr`, `jvirt_barray_ptr`

# code/jpeg-6/jquant1.c
## File Purpose
Implements 1-pass color quantization for JPEG decompression, mapping full-color pixel data to a fixed colormap using equally spaced color values. Supports three dithering modes: none, ordered (Bayer 16×16), and Floyd-Steinberg error diffusion.

## Core Responsibilities
- Build an orthogonal colormap by dividing available colors across output components
- Precompute a `colorindex` lookup table mapping pixel values to nearest colormap entries (premultiplied by stride)
- Provide fast per-row pixel quantization (general N-component and 3-component fast paths)
- Implement ordered dithering using a static Bayer matrix
- Implement Floyd-Steinberg dithering with per-component error accumulation arrays
- Initialize and configure the `jpeg_color_quantizer` vtable registered on `cinfo->cquantize`

## External Dependencies
- `jinclude.h` — platform includes, `SIZEOF`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` / `jerror.h` — JPEG structs, `GETJSAMPLE`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `ERREXIT*`, `TRACEMS*`, `jzero_far`
- `QUANT_1PASS_SUPPORTED` — compile-time guard; entire file is conditionally compiled
- `jzero_far` — defined elsewhere (jutils.c)
- `alloc_small`, `alloc_large`, `alloc_sarray` — provided by JPEG memory manager, defined elsewhere

# code/jpeg-6/jquant2.c
## File Purpose
Implements 2-pass color quantization (color mapping) for the IJG JPEG decompressor. Pass 1 builds a color usage histogram; pass 2 maps each pixel to the nearest entry in a custom colormap derived via median-cut, with optional Floyd-Steinberg dithering.

## Core Responsibilities
- Accumulate a 3D RGB histogram during prescan (pass 1)
- Run Heckbert/median-cut box-splitting to select a representative colormap
- Build a lazy-filled inverse colormap (histogram reused as lookup cache)
- Map pixels to colormap entries without dithering (`pass2_no_dither`)
- Map pixels to colormap entries with Floyd-Steinberg dithering (`pass2_fs_dither`)
- Initialize and own the error-limiting table for F-S dithering
- Register itself as the `cquantize` subobject on `j_decompress_ptr`

## External Dependencies
- `jinclude.h` — system includes, `SIZEOF`, `MEMZERO`
- `jpeglib.h` / `jpegint.h` / `jerror.h` — JPEG object definitions, `METHODDEF`, `LOCAL`, `GLOBAL`, `ERREXIT`, `TRACEMS1`, `jzero_far`, `RIGHT_SHIFT`, `SHIFT_TEMPS`, `GETJSAMPLE`, `MAXJSAMPLE`, `BITS_IN_JSAMPLE`
- `RGB_RED`, `RGB_GREEN`, `RGB_BLUE` — defined in `jmorecfg.h`; control component ordering macros
- `cinfo->mem->alloc_small/alloc_large/alloc_sarray` — defined in memory manager, called via function pointers

# code/jpeg-6/jutils.c
## File Purpose
Provides shared utility tables and helper functions used by both the JPEG compressor and decompressor. Contains the canonical DCT coefficient ordering tables and low-level memory copy/zero operations needed throughout the IJG JPEG library.

## Core Responsibilities
- Define the `jpeg_zigzag_order` table mapping natural-order DCT positions to zigzag positions
- Define the `jpeg_natural_order` table mapping zigzag positions back to natural order (with overflow-safe padding)
- Provide integer arithmetic helpers (`jdiv_round_up`, `jround_up`)
- Provide portable sample-array row copy (`jcopy_sample_rows`)
- Provide portable DCT coefficient block row copy (`jcopy_block_row`)
- Provide FAR-pointer-safe memory zeroing (`jzero_far`) for DOS/80x86 compatibility

## External Dependencies
- `jinclude.h` — provides `MEMCOPY`, `MEMZERO`, `SIZEOF`, platform include dispatch
- `jpeglib.h` — provides `JSAMPARRAY`, `JBLOCKROW`, `JCOEFPTR`, `JDIMENSION`, `DCTSIZE2`, `JCOEF`, `JSAMPLE`, `FAR`, `GLOBAL`
- `jconfig.h` / `jmorecfg.h` (via `jpeglib.h`) — provide `NEED_FAR_POINTERS`, `USE_FMEM`, type sizes
- `memcpy` / `memset` / `_fmemcpy` / `_fmemset` — defined in system `<string.h>` or DOS far-memory library

# code/jpeg-6/jversion.h
## File Purpose
Defines version and copyright identification macros for the Independent JPEG Group's (IJG) JPEG library version 6. It serves as the single authoritative source of version metadata for the library build.

## Core Responsibilities
- Declares the library version string (`JVERSION`)
- Declares the copyright notice string (`JCOPYRIGHT`)

## External Dependencies
- No includes.
- No external symbols.

---

**Notes:**
- `JVERSION` value is `"6  2-Aug-95"`, indicating JPEG library release 6, dated August 2, 1995.
- `JCOPYRIGHT` credits Thomas G. Lane and the Independent JPEG Group.
- This file is vendored into the Quake III Arena source tree as part of the embedded `jpeg-6` library used for JPEG texture loading (see `code/jpeg-6/jload.c`).

