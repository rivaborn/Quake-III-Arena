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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `savable_state` | struct | MCU-transient state (EOBRUN + per-component last DC value) that is only committed on successful MCU completion |
| `phuff_entropy_decoder` | struct | Full progressive entropy decoder object; extends `jpeg_entropy_decoder` with bitread state, savable state, restart counter, and derived Huffman tables |
| `phuff_entropy_ptr` | typedef | Pointer alias for `phuff_entropy_decoder*` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `extend_test` | `static const int[16]` | file-static | Threshold table for sign-extension (`HUFF_EXTEND`); entry n = 2^(n-1) |
| `extend_offset` | `static const int[16]` | file-static | Offset table for sign-extension; entry n = (-1<<n)+1 |

(Both are conditional on `!AVOID_TABLES`.)

## Key Functions / Methods

### start_pass_phuff_decoder
- **Signature:** `METHODDEF void start_pass_phuff_decoder(j_decompress_ptr cinfo)`
- **Purpose:** Called at the start of each progressive scan. Validates scan parameters, updates `coef_bits` progression table, selects the appropriate `decode_mcu` function pointer, builds derived Huffman tables, and resets all decoder state.
- **Inputs:** `cinfo` — decompressor context with scan parameters (Ss, Se, Ah, Al, comps_in_scan).
- **Outputs/Return:** void; sets `entropy->pub.decode_mcu` function pointer.
- **Side effects:** Allocates/updates derived Huffman tables in `entropy->derived_tbls[]`; writes `cinfo->coef_bits[][]`; resets bitstate and EOBRUN.
- **Calls:** `jpeg_make_d_derived_tbl`, `ERREXIT4`, `ERREXIT1`, `WARNMS2`.

### decode_mcu_DC_first
- **Signature:** `METHODDEF boolean decode_mcu_DC_first(j_decompress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Decodes one MCU for a DC initial scan. Reads Huffman-coded DC differences, applies delta-decoding, and stores left-shifted coefficient into block[0].
- **Inputs:** `cinfo`, `MCU_data` — array of block row pointers (pre-zeroed by caller).
- **Outputs/Return:** `TRUE` on success; `FALSE` on suspension (no permanent state change).
- **Side effects:** Writes `(*block)[0]` for each block; updates `entropy->bitstate` and `entropy->saved`; decrements `restarts_to_go`.
- **Calls:** `process_restart`, `HUFF_DECODE`, `HUFF_EXTEND` (macro), `BITREAD_LOAD_STATE`/`BITREAD_SAVE_STATE`, `ASSIGN_STATE`.

### decode_mcu_AC_first
- **Signature:** `METHODDEF boolean decode_mcu_AC_first(j_decompress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Decodes one MCU (single block) for an AC initial scan. Handles ZRL (run of 15 zeros), EOBr run-length codes, and individual AC coefficient values; dezigzags and bit-shifts output.
- **Inputs:** `cinfo`, `MCU_data`.
- **Outputs/Return:** `TRUE`/`FALSE` (suspension).
- **Side effects:** Writes AC coefficients to `*block` at natural-order positions; updates EOBRUN and bitstate.
- **Calls:** `process_restart`, `HUFF_DECODE`, `HUFF_EXTEND`, `jpeg_natural_order` (extern array).

### decode_mcu_DC_refine
- **Signature:** `METHODDEF boolean decode_mcu_DC_refine(j_decompress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Decodes DC successive approximation refinement bits. Each block reads exactly one raw bit and ORs it into block[0] at position Al.
- **Inputs:** `cinfo`, `MCU_data`.
- **Outputs/Return:** `TRUE`/`FALSE`.
- **Side effects:** Modifies `(*block)[0]` in-place; updates bitstate.
- **Calls:** `process_restart`, `CHECK_BIT_BUFFER`, `GET_BITS`.

### decode_mcu_AC_refine
- **Signature:** `METHODDEF boolean decode_mcu_AC_refine(j_decompress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Decodes AC successive approximation refinement. Applies correction bits to already-nonzero coefficients and assigns sign-only values to newly nonzero ones; tracks newly-set positions in `newnz_pos[]` to allow rollback on suspension.
- **Inputs:** `cinfo`, `MCU_data`.
- **Outputs/Return:** `TRUE`/`FALSE`. On `FALSE`, undoes all newly-nonzero assignments via `undoit` label.
- **Side effects:** Modifies AC coefficients in-place; updates EOBRUN and bitstate.
- **Calls:** `process_restart`, `HUFF_DECODE`, `CHECK_BIT_BUFFER`, `GET_BITS`, `WARNMS`.
- **Notes:** The `undoit` cleanup path zeroes `newnz_pos[]` entries — critical for idempotency across suspension retries.

### process_restart
- **Signature:** `LOCAL boolean process_restart(j_decompress_ptr cinfo)`
- **Purpose:** Consumes an RSTn marker, resets DC predictions, EOBRUN, restart counter, and EOD warning flag.
- **Inputs:** `cinfo`.
- **Outputs/Return:** `TRUE` on success; `FALSE` if marker read suspends.
- **Side effects:** Modifies `entropy->saved`, `entropy->restarts_to_go`, `entropy->bitstate.printed_eod`; increments `marker->discarded_bytes`.
- **Calls:** `cinfo->marker->read_restart_marker`.

### jinit_phuff_decoder
- **Signature:** `GLOBAL void jinit_phuff_decoder(j_decompress_ptr cinfo)`
- **Purpose:** Allocates and installs the `phuff_entropy_decoder` object; initializes derived table pointers to NULL; allocates and zeroes `cinfo->coef_bits` progression table.
- **Inputs:** `cinfo`.
- **Outputs/Return:** void.
- **Side effects:** Allocates memory via `cinfo->mem->alloc_small` (JPOOL_IMAGE); sets `cinfo->entropy` and `cinfo->coef_bits`.

## Control Flow Notes
- **Init:** `jinit_phuff_decoder` is called once during decompressor setup.
- **Per-scan:** `start_pass_phuff_decoder` is called at the start of each scan, routing to one of four `decode_mcu_*` variants.
- **Per-MCU (frame loop):** The selected `decode_mcu` is called repeatedly by the coefficient controller. On suspension, the function returns `FALSE` without updating permanent state; the caller retries the same MCU.
- **Restart:** Checked at the top of each `decode_mcu_*` when `restarts_to_go == 0`.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, etc.)
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JBLOCKROW`, `JCOEF`, scan parameter fields
- `jdhuff.h` — `d_derived_tbl`, `bitread_perm_state`, `bitread_working_state`, `HUFF_DECODE`, `CHECK_BIT_BUFFER`, `GET_BITS`, `BITREAD_*` macros
- **Defined elsewhere:** `jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`, `jpeg_natural_order`, `ERREXIT*`/`WARNMS*` error macros
