# code/jpeg-6/jchuff.h

## File Purpose
Declares shared data structures and function prototypes for JPEG Huffman entropy encoding, used by both the sequential encoder (`jchuff.c`) and the progressive encoder (`jcphuff.c`).

## Core Responsibilities
- Define the `c_derived_tbl` structure representing a pre-computed Huffman encoding table
- Declare `jpeg_make_c_derived_tbl` for expanding a raw Huffman table into derived (ready-to-use) form
- Declare `jpeg_gen_optimal_table` for generating an optimal Huffman table from symbol frequency data
- Provide short-name aliases for linkers with limited external symbol length support

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `c_derived_tbl` | struct | Holds pre-computed Huffman codes (`ehufco`) and code lengths (`ehufsi`) for all 256 symbols, ready for fast encoding |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_make_c_derived_tbl
- **Signature:** `EXTERN void jpeg_make_c_derived_tbl(j_compress_ptr cinfo, JHUFF_TBL *htbl, c_derived_tbl **pdtbl)`
- **Purpose:** Expands a raw `JHUFF_TBL` definition into a `c_derived_tbl` with precomputed symbol codes and lengths.
- **Inputs:** `cinfo` — compressor context; `htbl` — source Huffman table; `pdtbl` — output pointer to derived table.
- **Outputs/Return:** `void`; writes derived table through `pdtbl`.
- **Side effects:** Allocates memory for `c_derived_tbl` via JPEG memory manager (defined elsewhere).
- **Calls:** Not inferable from this file (declaration only).
- **Notes:** A zero value in `ehufsi[S]` indicates no code was assigned to symbol `S`.

### jpeg_gen_optimal_table
- **Signature:** `EXTERN void jpeg_gen_optimal_table(j_compress_ptr cinfo, JHUFF_TBL *htbl, long freq[])`
- **Purpose:** Generates an optimal (minimum-redundancy) Huffman table definition from a symbol frequency array.
- **Inputs:** `cinfo` — compressor context; `htbl` — output table to fill; `freq[]` — symbol frequency counts.
- **Outputs/Return:** `void`; writes resulting table into `htbl`.
- **Side effects:** Modifies `htbl` in place.
- **Calls:** Not inferable from this file (declaration only).
- **Notes:** Implements the standard Huffman tree construction algorithm driven by observed symbol frequencies.

## Control Flow Notes
This is a header included during JPEG compression initialization/encoding phases. `jpeg_gen_optimal_table` is typically called after a statistics-gathering pass, and `jpeg_make_c_derived_tbl` is called during encoder setup before the actual entropy-coding pass begins.

## External Dependencies
- `jpeglib.h` / `jpegint.h` — defines `j_compress_ptr`, `JHUFF_TBL`, `JPP`, `EXTERN`
- `jpeg_make_c_derived_tbl`, `jpeg_gen_optimal_table` — defined in `jchuff.c`
