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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `savable_state` | struct | Bit buffer + last DC values; snapshotted per MCU for suspension rollback |
| `huff_entropy_encoder` | struct | Full encoder object; extends `jpeg_entropy_encoder`; holds derived tables and statistics pointers |
| `working_state` | struct | Per-MCU working copy of output pointer, buffer space, and `savable_state`; passed through encoding helpers |
| `c_derived_tbl` | struct (from jchuff.h) | Flat arrays `ehufco[256]`/`ehufsi[256]` mapping symbol → code/size |

## Global / File-Static State

None.

## Key Functions / Methods

### `jinit_huff_encoder`
- Signature: `GLOBAL void jinit_huff_encoder(j_compress_ptr cinfo)`
- Purpose: Module init; allocates `huff_entropy_encoder`, wires `start_pass` method, NULLs all table pointers.
- Inputs: `cinfo` – active compression object
- Outputs/Return: void; sets `cinfo->entropy`
- Side effects: Allocates from `JPOOL_IMAGE`
- Calls: `alloc_small`
- Notes: Must be called before any compression pass.

### `start_pass_huff`
- Signature: `METHODDEF void start_pass_huff(j_compress_ptr cinfo, boolean gather_statistics)`
- Purpose: Prepares encoder for a scan; either builds derived tables (encoding mode) or allocates/zeroes count arrays (statistics mode).
- Inputs: `cinfo`, `gather_statistics` flag
- Outputs/Return: void
- Side effects: Sets `entropy->pub.encode_mcu` / `finish_pass` function pointers; resets bit buffer and DC predictions; allocates stat arrays if needed
- Calls: `jpeg_make_c_derived_tbl`, `alloc_small`, `MEMZERO`
- Notes: Errors if required Huffman tables are absent in encoding mode.

### `jpeg_make_c_derived_tbl`
- Signature: `GLOBAL void jpeg_make_c_derived_tbl(j_compress_ptr cinfo, JHUFF_TBL *htbl, c_derived_tbl **pdtbl)`
- Purpose: Converts a JPEG DHT-format table (`bits[]`/`huffval[]`) into the flat `ehufco`/`ehufsi` arrays used for fast symbol lookup during encoding.
- Inputs: `htbl` – source Huffman table; `pdtbl` – output pointer (allocated if NULL)
- Outputs/Return: void; fills `*pdtbl`
- Side effects: May allocate from `JPOOL_IMAGE`
- Calls: `alloc_small`, `MEMZERO`
- Notes: Also called by `jcphuff.c`. Symbols with no code get `ehufsi[s]=0`, which `emit_bits` treats as an error.

### `encode_mcu_huff`
- Signature: `METHODDEF boolean encode_mcu_huff(j_compress_ptr cinfo, JBLOCKROW *MCU_data)`
- Purpose: Encodes one MCU; suspendable — copies state to `working_state`, encodes all blocks, then commits state back only on success.
- Inputs: `cinfo`, `MCU_data` – array of DCT blocks
- Outputs/Return: TRUE on success, FALSE if output suspension required
- Side effects: Writes bytes to `cinfo->dest`; updates `entropy->saved`, `restarts_to_go`, `next_restart_num`
- Calls: `emit_restart`, `encode_one_block`
- Notes: Core per-MCU entry point during the encoding pass.

### `encode_one_block`
- Signature: `LOCAL boolean encode_one_block(working_state *state, JCOEFPTR block, int last_dc_val, c_derived_tbl *dctbl, c_derived_tbl *actbl)`
- Purpose: Encodes one 8×8 DCT block: DC difference then AC run-length/amplitude pairs per JPEG Section F.1.2.
- Inputs: block coefficients, previous DC value, DC and AC derived tables
- Outputs/Return: TRUE/FALSE (suspension)
- Side effects: Calls `emit_bits` which writes to output buffer
- Calls: `emit_bits`
- Notes: Uses `jpeg_natural_order` (defined elsewhere) for zigzag traversal. ZRL symbol `0xF0` emitted for runs > 15; EOB `0x00` for trailing zeros.

### `emit_bits`
- Signature: `INLINE LOCAL boolean emit_bits(working_state *state, unsigned int code, int size)`
- Purpose: Shifts `code` (up to 16 bits) into a 24-bit accumulator and flushes whole bytes, stuffing `0x00` after any `0xFF` byte per JPEG spec.
- Inputs: code value, bit width
- Outputs/Return: TRUE/FALSE (suspension)
- Side effects: Writes to `state->next_output_byte`; may call `dump_buffer`
- Calls: `dump_buffer` (via `emit_byte` macro)
- Notes: Critical hot path; size==0 is a fatal error.

### `jpeg_gen_optimal_table`
- Signature: `GLOBAL void jpeg_gen_optimal_table(j_compress_ptr cinfo, JHUFF_TBL *htbl, long freq[])`
- Purpose: Implements Huffman's algorithm (JPEG Annex K.2) over 257 symbols, then adjusts code lengths to the JPEG 16-bit maximum, and fills `htbl->bits`/`huffval`.
- Inputs: `freq[]` — symbol usage counts (257 entries)
- Outputs/Return: void; updates `htbl`; sets `htbl->sent_table = FALSE`
- Side effects: Modifies `freq[]` in place (clobbers counts)
- Calls: `MEMZERO`, `MEMCOPY`, `ERREXIT`
- Notes: Also used by `jcphuff.c`. The pseudo-symbol 256 ensures no real symbol gets an all-ones code.

## Control Flow Notes
- **Init**: `jinit_huff_encoder` called once during compressor setup.
- **Per-scan**: `start_pass_huff` (or `start_pass_huff` with `gather_statistics=TRUE`) called before the scan loop.
- **Per-MCU**: `encode_mcu_huff` (or `encode_mcu_gather`) called by the coefficient controller for each MCU row.
- **End of scan**: `finish_pass_huff` flushes the bit buffer; `finish_pass_gather` generates optimal tables from accumulated counts.
- The two-pass flow (gather → regenerate tables → encode) is gated on `ENTROPY_OPT_SUPPORTED`.

## External Dependencies
- `jinclude.h` — portability macros (`MEMZERO`, `MEMCOPY`, `SIZEOF`)
- `jpeglib.h` — `j_compress_ptr`, `jpeg_entropy_encoder`, `JHUFF_TBL`, `jpeg_component_info`, `jpeg_destination_mgr`
- `jchuff.h` — `c_derived_tbl`, shared extern declarations for `jpeg_make_c_derived_tbl` / `jpeg_gen_optimal_table`
- **Defined elsewhere**: `jpeg_natural_order` (zigzag index table, `jpegint.h`/`jutils.c`), `jpeg_alloc_huff_table`, all JPEG error codes (`JERR_*`), `ERREXIT`/`ERREXIT1` macros
