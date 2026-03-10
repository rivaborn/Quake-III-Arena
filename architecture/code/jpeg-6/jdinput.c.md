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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `my_input_controller` | struct | Private extension of `jpeg_input_controller`; adds `inheaders` flag tracking pre-first-SOS state |
| `my_inputctl_ptr` | typedef | Pointer alias for `my_input_controller` |

## Global / File-Static State
None.

## Key Functions / Methods

### initial_setup
- **Signature:** `LOCAL void initial_setup(j_decompress_ptr cinfo)`
- **Purpose:** One-time validation and geometry computation triggered by the first SOS marker.
- **Inputs:** `cinfo` — decompressor state with image header already parsed
- **Outputs/Return:** void; populates `cinfo->comp_info[*]` dimensions, `max_h/v_samp_factor`, `total_iMCU_rows`, `has_multiple_scans`
- **Side effects:** Writes to multiple `cinfo` fields; calls `ERREXIT` on constraint violations
- **Calls:** `jdiv_round_up`, `ERREXIT`/`ERREXIT1`/`ERREXIT2`
- **Notes:** `DCT_scaled_size` is set to `DCTSIZE` here as a fallback for the transcoder path; `jdmaster.c` overrides it in full decompression.

### per_scan_setup
- **Signature:** `LOCAL void per_scan_setup(j_decompress_ptr cinfo)`
- **Purpose:** Computes MCU geometry for each scan based on interleaving mode.
- **Inputs:** `cinfo->comps_in_scan`, `cinfo->cur_comp_info[]`
- **Outputs/Return:** void; sets `MCUs_per_row`, `MCU_rows_in_scan`, per-component `MCU_width/height/blocks`, `last_col/row_width/height`, `MCU_membership[]`
- **Side effects:** Writes `cinfo` scan fields
- **Calls:** `jdiv_round_up`, `ERREXIT`/`ERREXIT2`
- **Notes:** Non-interleaved path always sets `MCU_width = MCU_height = 1`.

### latch_quant_tables
- **Signature:** `LOCAL void latch_quant_tables(j_decompress_ptr cinfo)`
- **Purpose:** Copies the current quantization table for each scan component into `compptr->quant_table` if not already saved, enabling correct dequantization in multi-scan files where tables may be redefined between scans.
- **Inputs:** `cinfo->cur_comp_info[]`, `cinfo->quant_tbl_ptrs[]`
- **Outputs/Return:** void; allocates and fills `compptr->quant_table` from `JPOOL_IMAGE`
- **Side effects:** Allocates memory via `cinfo->mem->alloc_small`
- **Calls:** `MEMCOPY`, `ERREXIT1`

### start_input_pass
- **Signature:** `METHODDEF void start_input_pass(j_decompress_ptr cinfo)`
- **Purpose:** Initializes all input-side submodules for a new scan and switches `consume_input` to the coefficient controller's data consumer.
- **Inputs:** `cinfo`
- **Outputs/Return:** void
- **Side effects:** Calls `per_scan_setup`, `latch_quant_tables`; invokes `entropy->start_pass`, `coef->start_input_pass`; sets `inputctl->consume_input = cinfo->coef->consume_data`

### finish_input_pass
- **Signature:** `METHODDEF void finish_input_pass(j_decompress_ptr cinfo)`
- **Purpose:** Reverts `consume_input` back to `consume_markers` after all expected compressed data in a scan has been read.
- **Side effects:** Resets `inputctl->consume_input` function pointer

### consume_markers
- **Signature:** `METHODDEF int consume_markers(j_decompress_ptr cinfo)`
- **Purpose:** The marker-phase `consume_input` implementation; reads markers and transitions state on SOS/EOI events.
- **Inputs:** `cinfo`
- **Outputs/Return:** `JPEG_SUSPENDED`, `JPEG_REACHED_SOS`, or `JPEG_REACHED_EOI`
- **Side effects:** May call `initial_setup` or `start_input_pass`; sets `eoi_reached`; adjusts `output_scan_number` on EOI
- **Calls:** `cinfo->marker->read_markers`, `initial_setup`, `start_input_pass`, `ERREXIT`

### reset_input_controller
- **Signature:** `METHODDEF void reset_input_controller(j_decompress_ptr cinfo)`
- **Purpose:** Resets all input controller state to begin a fresh datastream, also resets error manager and marker reader.
- **Side effects:** Writes `inputctl` flags; calls `err->reset_error_mgr`, `marker->reset_marker_reader`; sets `cinfo->coef_bits = NULL`

### jinit_input_controller
- **Signature:** `GLOBAL void jinit_input_controller(j_decompress_ptr cinfo)`
- **Purpose:** Allocates and installs the `my_input_controller` subobject, wires method pointers.
- **Side effects:** Allocates from `JPOOL_PERMANENT`; sets `cinfo->inputctl`

## Control Flow Notes
Called once during decompressor creation (`jinit_input_controller`). During decompression, `consume_input` is the main polling entry point — it alternates between `consume_markers` (inter-scan state) and `coef->consume_data` (intra-scan data reading). The first SOS triggers `initial_setup`; each subsequent SOS triggers `start_input_pass`. EOI sets `eoi_reached` and short-circuits further calls.

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMCOPY`, `SIZEOF`, etc.)
- `jpeglib.h` / `jpegint.h` — all JPEG types and submodule interfaces
- **Defined elsewhere:** `jdiv_round_up` (jutils.c), `cinfo->marker->read_markers` (jdmarker.c), `cinfo->entropy->start_pass` (jdhuff.c / jdphuff.c), `cinfo->coef->start_input_pass` / `consume_data` (jdcoefct.c), `cinfo->mem->alloc_small` (jmemmgr.c)
