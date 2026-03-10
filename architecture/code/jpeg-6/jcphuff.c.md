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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `phuff_entropy_encoder` | struct | Extended entropy encoder holding bit buffer, EOBRUN state, correction-bit buffer, derived/stats tables, and restart state |
| `phuff_entropy_ptr` | typedef | Pointer alias to `phuff_entropy_encoder` |
| `c_derived_tbl` | typedef (struct, from jchuff.h) | Precomputed Huffman code/length arrays indexed by symbol |

## Global / File-Static State

None.

## Key Functions / Methods

### jinit_phuff_encoder
- **Signature:** `GLOBAL void jinit_phuff_encoder(j_compress_ptr cinfo)`
- **Purpose:** Module init entry point; allocates the encoder object and registers `start_pass_phuff`.
- **Inputs:** `cinfo` — compression context
- **Outputs/Return:** void; sets `cinfo->entropy`
- **Side effects:** Allocates `phuff_entropy_encoder` from JPOOL_IMAGE; nulls all table and bit-buffer pointers.
- **Calls:** `cinfo->mem->alloc_small`
- **Notes:** Called once per compression instance during setup.

### start_pass_phuff
- **Signature:** `METHODDEF void start_pass_phuff(j_compress_ptr cinfo, boolean gather_statistics)`
- **Purpose:** Per-scan initialization; selects encode-MCU and finish-pass function pointers based on scan type (DC/AC, first/refine, stats/output).
- **Inputs:** `cinfo`, `gather_statistics` flag
- **Outputs/Return:** void
- **Side effects:** Allocates correction-bit buffer for AC refinement if needed; allocates/zeroes statistics arrays or builds derived tables; resets EOBRUN, bit buffer, restart counters.
- **Calls:** `jpeg_make_c_derived_tbl`, `cinfo->mem->alloc_small`, `MEMZERO`, `ERREXIT1`
- **Notes:** DC refinement requires no Huffman table. AC only ever has one component per scan (`comps_in_scan == 1`).

### encode_mcu_DC_first
- **Signature:** `METHODDEF boolean encode_mcu_DC_first(j_compress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Encodes DC coefficients for the initial (first) scan pass, applying point transform `Al` and DPCM differential coding.
- **Inputs:** `cinfo`, `MCU_data` block rows
- **Outputs/Return:** `TRUE` always (no suspension)
- **Side effects:** Writes Huffman-coded bits to output buffer; updates `last_dc_val[]`; handles restart markers.
- **Calls:** `emit_restart`, `emit_symbol`, `emit_bits`
- **Notes:** Uses `IRIGHT_SHIFT` for arithmetic right shift portability.

### encode_mcu_AC_first
- **Signature:** `METHODDEF boolean encode_mcu_AC_first(j_compress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Encodes AC coefficients for the initial scan, with zero run-length coding and deferred EOB-run accumulation.
- **Inputs:** `cinfo`, `MCU_data`
- **Outputs/Return:** `TRUE` always
- **Side effects:** Accumulates `EOBRUN`; emits ZRL (0xF0) codes for runs > 15; forces EOBRUN flush at 0x7FFF to prevent overflow.
- **Calls:** `emit_restart`, `emit_eobrun`, `emit_symbol`, `emit_bits`
- **Notes:** Point transform rounds AC values toward zero via shift after abs().

### encode_mcu_DC_refine
- **Signature:** `METHODDEF boolean encode_mcu_DC_refine(j_compress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** Refinement pass for DC: emits a single raw bit (the `Al`-th bit) per block, no Huffman coding needed.
- **Inputs:** `cinfo`, `MCU_data`
- **Outputs/Return:** `TRUE` always
- **Side effects:** Emits one bit per DCT block; handles restart markers.
- **Calls:** `emit_restart`, `emit_bits`

### encode_mcu_AC_refine
- **Signature:** `METHODDEF boolean encode_mcu_AC_refine(j_compress_ptr cinfo, JBLOCKROW *MCU_data)`
- **Purpose:** AC successive-approximation refinement; pre-computes absolute values, buffers correction bits, emits ZRL/EOB symbols per spec section G.1.2.3.
- **Inputs:** `cinfo`, `MCU_data`
- **Outputs/Return:** `TRUE` always
- **Side effects:** Appends to `bit_buffer`/`BE`; flushes EOBRUN when overflow thresholds are approached.
- **Calls:** `emit_restart`, `emit_eobrun`, `emit_symbol`, `emit_bits`, `emit_buffered_bits`
- **Notes:** Uses a local `absvalues[]` pre-pass to find EOB position efficiently.

### finish_pass_phuff / finish_pass_gather_phuff
- Flush pending EOBRUN and bit buffer on output pass; on stats pass, call `jpeg_gen_optimal_table` once per unique table to produce optimal Huffman tables.

### emit_bits / emit_symbol / emit_eobrun / emit_buffered_bits / emit_restart / flush_bits / dump_buffer
- Low-level bit/byte output helpers; `emit_bits` packs bits left-justified in a 24-bit window and byte-stuffs 0xFF→0xFF00.

## Control Flow Notes
Called during the JPEG compression pipeline's entropy-coding stage. `jinit_phuff_encoder` runs at startup; `start_pass_phuff` is called before each scan; the selected `encode_mcu` variant is invoked per MCU row by the coefficient controller; `finish_pass` is called at scan end. Two-pass optimization requires a statistics gather pass before the actual output pass.

## External Dependencies
- `jinclude.h` — platform portability macros (`MEMZERO`, `SIZEOF`, etc.)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_component_info`, `JHUFF_TBL`, `jpeg_destination_mgr`, `JBLOCKROW`, scan params (`Ss`, `Se`, `Ah`, `Al`)
- `jchuff.h` — `c_derived_tbl`, `jpeg_make_c_derived_tbl`, `jpeg_gen_optimal_table`
- **Defined elsewhere:** `jpeg_natural_order` (zigzag-to-natural scan order table), `jpeg_alloc_huff_table`, `JPEG_RST0`, `JERR_*` error codes, `ERREXIT`/`ERREXIT1` macros
