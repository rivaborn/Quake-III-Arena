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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `savable_state` | struct | Per-component last DC value; saved/restored around MCU decode for suspension support |
| `huff_entropy_decoder` | struct | Full entropy decoder state: bitread state, savable state, restart counter, derived DC/AC tables |
| `d_derived_tbl` | struct (jdhuff.h) | Precomputed decode tables: mincode/maxcode/valptr arrays + 8-bit lookahead tables |
| `bitread_perm_state` | struct (jdhuff.h) | Persistent bit-buffer state saved between MCUs |
| `bitread_working_state` | struct (jdhuff.h) | Local working copy of bit-buffer state used during MCU decode |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `extend_test` | `static const int[16]` | file-static | Threshold table for sign-extension (`HUFF_EXTEND`); 2^(n-1) per entry |
| `extend_offset` | `static const int[16]` | file-static | Addend table for sign-extension; `(-1<<n)+1` per entry |

## Key Functions / Methods

### `jinit_huff_decoder`
- Signature: `GLOBAL void jinit_huff_decoder(j_decompress_ptr cinfo)`
- Purpose: Allocates `huff_entropy_decoder`, sets method pointers, nulls derived table pointers.
- Inputs: `cinfo` — decompress master record
- Outputs/Return: void; sets `cinfo->entropy`
- Side effects: Allocates from `JPOOL_IMAGE` via JPEG memory manager
- Calls: `cinfo->mem->alloc_small`
- Notes: Called once per image during decompressor initialization.

### `start_pass_huff_decoder`
- Signature: `METHODDEF void start_pass_huff_decoder(j_decompress_ptr cinfo)`
- Purpose: Validates sequential scan parameters, builds derived Huffman tables for all scan components, resets DC predictors and bit-reader state.
- Inputs: `cinfo` with scan parameters and Huffman table pointers populated
- Outputs/Return: void
- Side effects: May emit `JWRN_NOT_SEQUENTIAL` warning; calls `jpeg_make_d_derived_tbl` which may allocate
- Calls: `jpeg_make_d_derived_tbl`, `WARNMS`, `ERREXIT1`

### `jpeg_make_d_derived_tbl`
- Signature: `GLOBAL void jpeg_make_d_derived_tbl(j_decompress_ptr cinfo, JHUFF_TBL *htbl, d_derived_tbl **pdtbl)`
- Purpose: Generates `d_derived_tbl` from a raw `JHUFF_TBL`: computes canonical codes (Figures C.1/C.2), fills min/max/valptr arrays (Figure F.15), populates 8-bit lookahead tables.
- Inputs: `htbl` — raw bits/huffval table; `pdtbl` — output pointer (allocates if NULL)
- Outputs/Return: void; `*pdtbl` populated
- Side effects: Allocates `d_derived_tbl` from `JPOOL_IMAGE` on first call
- Notes: Also called by `jdphuff.c` (shared via GLOBAL linkage).

### `jpeg_fill_bit_buffer`
- Signature: `GLOBAL boolean jpeg_fill_bit_buffer(bitread_working_state *state, register bit_buf_type get_buffer, register int bits_left, int nbits)`
- Purpose: Refills `get_buffer` to at least `MIN_GET_BITS` bits by reading bytes from the source, handling 0xFF byte stuffing and JPEG markers.
- Inputs: Working bit state, current buffer/count, minimum bits needed
- Outputs/Return: `TRUE` on success (state updated); `FALSE` if source suspended
- Side effects: Advances `next_input_byte`/`bytes_in_buffer`; may call `fill_input_buffer`; sets `unread_marker`; emits `JWRN_HIT_MARKER` once per segment
- Notes: On marker hit with sufficient bits remaining, inserts zero bytes rather than failing.

### `jpeg_huff_decode`
- Signature: `GLOBAL int jpeg_huff_decode(bitread_working_state *state, register bit_buf_type get_buffer, register int bits_left, d_derived_tbl *htbl, int min_bits)`
- Purpose: Slow-path Huffman decode: fetches bits one at a time until code matches `maxcode[l]`, then looks up symbol in `huffval`.
- Inputs: Bit state, starting code length `min_bits`, derived table
- Outputs/Return: Decoded symbol (0–255), or `-1` on suspension, or `0` on bad code (with warning)
- Calls: `CHECK_BIT_BUFFER` (macro → `jpeg_fill_bit_buffer`), `WARNMS`

### `decode_mcu`
- Signature: `METHODDEF boolean decode_mcu(j_decompress_ptr cinfo, JBLOCKROW *MCU_data)`
- Purpose: Decodes one MCU: for each block, decodes DC difference + DPCM accumulation, then AC run-length coefficients in zigzag order, writing to caller-zeroed `JBLOCKROW` output.
- Inputs: `cinfo` with entropy state; `MCU_data` array of pre-zeroed coefficient blocks
- Outputs/Return: `TRUE` on success; `FALSE` if source suspended (no permanent state changed)
- Side effects: Updates `entropy->bitstate`, `entropy->saved`, `entropy->restarts_to_go`; calls `process_restart` if needed
- Notes: Uses `ASSIGN_STATE` copy-on-success idiom. Skips AC decode when `component_needed` is false or `DCT_scaled_size == 1`.

### `process_restart`
- Signature: `LOCAL boolean process_restart(j_decompress_ptr cinfo)`
- Purpose: Consumes a restart marker, resets DC predictions to 0, resets restart counter and EOD warning flag.
- Inputs: `cinfo` with entropy and marker state
- Outputs/Return: `TRUE` on success; `FALSE` if marker read suspended
- Side effects: Flushes partial byte from bit buffer (counted as discarded), calls `read_restart_marker`

## Control Flow Notes
- **Init**: `jinit_huff_decoder` called during decompressor startup.
- **Per-scan**: `start_pass_huff_decoder` called before each scan begins; builds tables.
- **Per-MCU**: `decode_mcu` called by the coefficient controller for every MCU in the scan. It snapshots state locally, decodes, and only commits state on full success — enabling clean suspension/resume.
- No render or shutdown involvement; purely a data-processing stage in the decompression pipeline.

## External Dependencies
- `jinclude.h` — platform includes, `MEMZERO`, `SIZEOF`
- `jpeglib.h` — `j_decompress_ptr`, `jpeg_component_info`, `JHUFF_TBL`, `JBLOCKROW`, `JCOEF`
- `jdhuff.h` — `d_derived_tbl`, `bitread_*` types, `HUFF_DECODE`/`CHECK_BIT_BUFFER`/`GET_BITS` macros; shared with `jdphuff.c`
- `jpegint.h` (via `jpeglib.h` with `JPEG_INTERNALS`) — internal module structs
- `jerror.h` — `WARNMS`, `ERREXIT1`, warning/error codes
- **Defined elsewhere**: `jpeg_natural_order[]` (dezigzag table, used in `decode_mcu`); `cinfo->marker->read_restart_marker`; `cinfo->src->fill_input_buffer`
