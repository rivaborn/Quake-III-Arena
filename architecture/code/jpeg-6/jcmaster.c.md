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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `c_pass_type` | enum | Distinguishes `main_pass`, `huff_opt_pass`, and `output_pass` phases |
| `my_comp_master` | struct | Private master state; extends `jpeg_comp_master` with pass/scan counters and `pass_type` |
| `my_master_ptr` | typedef | Convenience pointer alias for `my_comp_master *` |

## Global / File-Static State
None.

## Key Functions / Methods

### initial_setup
- **Signature:** `LOCAL void initial_setup(j_compress_ptr cinfo)`
- **Purpose:** Pre-master-selection validation and geometry computation. Checks image dimensions, precision, component count, sampling factors, then computes `width_in_blocks`, `height_in_blocks`, `downsampled_width/height`, and `total_iMCU_rows` for all components.
- **Inputs:** `cinfo` with caller-filled image description fields.
- **Outputs/Return:** void; writes derived fields into `cinfo` and each `comp_info[]` entry.
- **Side effects:** Calls `ERREXIT` macros (longjmp-based) on any validation failure.
- **Calls:** `jdiv_round_up`, `ERREXIT`, `ERREXIT1`, `ERREXIT2`, `MAX`.
- **Notes:** DCT scaling is always 1× for compression (`DCT_scaled_size = DCTSIZE`).

### validate_script
- **Signature:** `LOCAL void validate_script(j_compress_ptr cinfo)` *(guarded by `C_MULTISCAN_FILES_SUPPORTED`)*
- **Purpose:** Validates the `cinfo->scan_info[]` multi-scan script; determines and sets `cinfo->progressive_mode`. Enforces JPEG standard rules for spectral selection (Ss/Se) and successive approximation (Ah/Al) ordering.
- **Inputs:** `cinfo->scan_info`, `cinfo->num_scans`, `cinfo->num_components`.
- **Outputs/Return:** void; sets `cinfo->progressive_mode`.
- **Side effects:** `ERREXIT` on any script violation.
- **Calls:** `ERREXIT`, `ERREXIT1`, `ERREXIT2`.
- **Notes:** Tracks `last_bitpos[component][coeff]` to verify AC scans are properly ordered; requires at least one DC scan per component in progressive mode.

### select_scan_parameters
- **Signature:** `LOCAL void select_scan_parameters(j_compress_ptr cinfo)`
- **Purpose:** Loads `cinfo->comps_in_scan`, `cur_comp_info[]`, `Ss/Se/Ah/Al` from the current scan script entry (or defaults for a single sequential scan).
- **Inputs:** `master->scan_number`, `cinfo->scan_info`.
- **Outputs/Return:** void; modifies `cinfo` scan-level fields.
- **Side effects:** none beyond `cinfo` state.
- **Calls:** `ERREXIT2`.

### per_scan_setup
- **Signature:** `LOCAL void per_scan_setup(j_compress_ptr cinfo)`
- **Purpose:** Computes MCU geometry for the current scan: `MCUs_per_row`, `MCU_rows_in_scan`, per-component `MCU_width/height/blocks`, `last_col_width`, `last_row_height`, `MCU_membership[]`, and `restart_interval`.
- **Inputs:** `cinfo->comps_in_scan`, `cur_comp_info[]`, image dimensions.
- **Outputs/Return:** void; writes into `cinfo` and each active `compptr`.
- **Side effects:** `ERREXIT` on MCU block count overflow.
- **Calls:** `jdiv_round_up`, `ERREXIT2`, `ERREXIT`, `MIN`.

### prepare_for_pass
- **Signature:** `METHODDEF void prepare_for_pass(j_compress_ptr cinfo)`
- **Purpose:** Called at the start of each pass. Dispatches `start_pass` to all relevant submodule objects based on `master->pass_type`; sets `call_pass_startup` and `is_last_pass` flags; updates progress monitor.
- **Inputs:** `master->pass_type`, `cinfo->optimize_coding`, `cinfo->raw_data_in`.
- **Outputs/Return:** void.
- **Side effects:** Calls `start_pass` on `cconvert`, `downsample`, `prep`, `fdct`, `entropy`, `coef`, `main`, and `marker` submodules depending on pass type.
- **Calls:** `select_scan_parameters`, `per_scan_setup`, all submodule `start_pass`/`write_*_header` methods.
- **Notes:** In `huff_opt_pass`, DC refinement scans fall through to `output_pass` (no optimization needed).

### pass_startup
- **Signature:** `METHODDEF void pass_startup(j_compress_ptr cinfo)`
- **Purpose:** One-shot hook called by `jpeg_write_scanlines` on first call in a single-pass compression to write frame and scan headers after the application has had a chance to write COM markers.
- **Inputs:** none beyond `cinfo`.
- **Outputs/Return:** void.
- **Side effects:** Writes JPEG frame/scan headers; resets `call_pass_startup = FALSE`.
- **Calls:** `cinfo->marker->write_frame_header`, `cinfo->marker->write_scan_header`.

### finish_pass_master
- **Signature:** `METHODDEF void finish_pass_master(j_compress_ptr cinfo)`
- **Purpose:** Finalizes each pass: flushes the entropy coder and advances `pass_type`, `scan_number`, and `pass_number` for the next pass.
- **Inputs:** `master->pass_type`, `cinfo->optimize_coding`.
- **Outputs/Return:** void.
- **Side effects:** Calls `entropy->finish_pass`; mutates master state.
- **Calls:** `cinfo->entropy->finish_pass`.

### jinit_c_master_control
- **Signature:** `GLOBAL void jinit_c_master_control(j_compress_ptr cinfo, boolean transcode_only)`
- **Purpose:** Allocates and initializes `my_comp_master`; wires it into `cinfo->master`; runs `initial_setup` and `validate_script`; computes `total_passes`.
- **Inputs:** `cinfo` (fully configured by caller), `transcode_only` flag.
- **Outputs/Return:** void; populates `cinfo->master`.
- **Side effects:** Allocates from `JPOOL_IMAGE`; forces `optimize_coding = TRUE` for progressive mode.
- **Calls:** `cinfo->mem->alloc_small`, `initial_setup`, `validate_script`, `ERREXIT`.
- **Notes:** `total_passes = num_scans * 2` when Huffman optimization is enabled; starting `pass_type` differs between transcode and normal paths.

## Control Flow Notes
`jinit_c_master_control` is called during `jpeg_start_compress` (init phase). Thereafter, the outer compression loop calls `prepare_for_pass` → processes data → `finish_pass_master` repeatedly until `is_last_pass` is true. `pass_startup` is invoked lazily on the first scanline write in single-pass mode.

## External Dependencies
- `jinclude.h` — system include resolution, `SIZEOF`, `MEMCOPY` macros
- `jpeglib.h` / `jpegint.h` (via `JPEG_INTERNALS`) — all JPEG struct definitions, `JMETHOD`, `JPOOL_IMAGE`, `DCTSIZE`, `DCTSIZE2`, `MAX_COMPONENTS`, `MAX_COMPS_IN_SCAN`, `C_MAX_BLOCKS_IN_MCU`
- `jerror.h` — `ERREXIT`, `ERREXIT1`, `ERREXIT2` macros and error codes
- `jdiv_round_up` — defined elsewhere (jutils.c); integer ceiling division
- All submodule vtable objects (`cconvert`, `downsample`, `prep`, `fdct`, `entropy`, `coef`, `main`, `marker`) — defined and initialized in their respective source files
