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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `d_derived_tbl` | struct | Pre-expanded Huffman table with `mincode`/`maxcode`/`valptr` arrays for canonical decode and an 8-bit lookahead cache (`look_nbits`, `look_sym`) |
| `bit_buf_type` | typedef (`INT32`) | Type of the 32-bit shift register used as the bit-extraction buffer |
| `bitread_perm_state` | struct | Persistent bitreader state saved across MCU boundaries (`get_buffer`, `bits_left`, `printed_eod`) |
| `bitread_working_state` | struct | Per-MCU working state held during decode; includes source pointer, byte count, unread marker, and back-link to `j_decompress_ptr` |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_make_d_derived_tbl
- Signature: `void jpeg_make_d_derived_tbl(j_decompress_ptr cinfo, JHUFF_TBL *htbl, d_derived_tbl **pdtbl)`
- Purpose: Expands a raw `JHUFF_TBL` (as parsed from the JPEG bitstream) into a `d_derived_tbl` with canonical min/max code arrays and the 8-bit lookahead cache.
- Inputs: Decompressor context, raw Huffman table, pointer-to-pointer to output derived table.
- Outputs/Return: void; allocates and populates `*pdtbl`.
- Side effects: Allocates memory via JPEG pool allocator.
- Calls: Defined in `jdhuff.c`; declaration only here.
- Notes: Must be called before any `HUFF_DECODE` use on a given table.

### jpeg_fill_bit_buffer
- Signature: `boolean jpeg_fill_bit_buffer(bitread_working_state *state, register bit_buf_type get_buffer, register int bits_left, int nbits)`
- Purpose: Out-of-line refill for the bit-extraction buffer when fewer than `nbits` bits remain; prefetches as many bytes as possible to amortize call overhead.
- Inputs: Working state (source pointer, byte count), current buffer contents, current bit count, minimum bits needed.
- Outputs/Return: `TRUE` on success (buffer has ≥ `nbits`); `FALSE` on suspension (insufficient data available).
- Side effects: Advances `state->next_input_byte`, decrements `state->bytes_in_buffer`, may set `state->unread_marker`.
- Calls: JPEG source manager `fill_input_buffer`.
- Notes: Inserts dummy zero bytes at EOI to allow graceful termination.

### jpeg_huff_decode
- Signature: `int jpeg_huff_decode(bitread_working_state *state, register bit_buf_type get_buffer, register int bits_left, d_derived_tbl *htbl, int min_bits)`
- Purpose: Slow-path Huffman decode for codes longer than `HUFF_LOOKAHEAD` (8) bits; used as fallback from the `HUFF_DECODE` macro.
- Inputs: Working bitread state, current buffer, bit count, derived table, minimum code length to start searching from.
- Outputs/Return: Decoded symbol value (0–255), or `-1` on suspension.
- Side effects: Updates `state->get_buffer` and `state->bits_left`.
- Calls: May internally call `jpeg_fill_bit_buffer`.
- Notes: Handles codes up to 16 bits per JPEG spec; `maxcode[17]` sentinel ensures termination.

## Control Flow Notes
This header is purely a shared interface—no frame/update loop here. At decode time: `jdhuff.c` and `jdphuff.c` call `jpeg_make_d_derived_tbl` during decoder initialization, then use `BITREAD_LOAD_STATE` to pull persistent state into registers at the start of each MCU, run `HUFF_DECODE` per coefficient (hitting the fast lookahead path ~95% of the time), and flush back with `BITREAD_SAVE_STATE` to support suspension between MCUs.

## External Dependencies
- `jpeglib.h` — `j_decompress_ptr`, `JHUFF_TBL`, `JOCTET`, `boolean`, `INT32`, `UINT8`, `JPP()`
- `jdhuff.c` — defines `jpeg_make_d_derived_tbl`, `jpeg_fill_bit_buffer`, `jpeg_huff_decode`
- `jdphuff.c` — consumer of all three extern functions and all macros defined here
