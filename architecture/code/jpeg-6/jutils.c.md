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

## Key Types / Data Structures
None (file is purely tables and utility functions; all types come from `jpeglib.h`/`jmorecfg.h`).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `jpeg_zigzag_order` | `const int[64]` | global | Maps natural-order index → zigzag position for a DCT 8×8 block |
| `jpeg_natural_order` | `const int[80]` | global | Maps zigzag index → natural-order position; 16 extra `63` entries guard against corrupt run-length overread in the Huffman decoder |

## Key Functions / Methods

### jdiv_round_up
- **Signature:** `GLOBAL long jdiv_round_up(long a, long b)`
- **Purpose:** Ceiling integer division: ⌈a/b⌉
- **Inputs:** `a` ≥ 0, `b` > 0
- **Outputs/Return:** `long` — ceiling quotient
- **Side effects:** None
- **Calls:** None
- **Notes:** Assumes non-negative `a`; used for dimension calculations throughout the library

### jround_up
- **Signature:** `GLOBAL long jround_up(long a, long b)`
- **Purpose:** Rounds `a` up to the next multiple of `b`
- **Inputs:** `a` ≥ 0, `b` > 0
- **Outputs/Return:** `long` — rounded value
- **Side effects:** None
- **Calls:** None

### jcopy_sample_rows
- **Signature:** `GLOBAL void jcopy_sample_rows(JSAMPARRAY input_array, int source_row, JSAMPARRAY output_array, int dest_row, int num_rows, JDIMENSION num_cols)`
- **Purpose:** Copies `num_rows` rows of pixel samples between two `JSAMPARRAY` buffers; supports overlapping regions
- **Inputs:** Source/dest arrays with row offsets, row count, column count
- **Outputs/Return:** void; writes to `output_array`
- **Side effects:** Memory write to output buffer
- **Calls:** `FMEMCOPY` (resolves to `MEMCOPY`/`memcpy` on non-80x86, or `_fmemcpy` on DOS FAR model)
- **Notes:** Falls back to a byte loop when `FMEMCOPY` is unavailable (no `USE_FMEM` on 80x86)

### jcopy_block_row
- **Signature:** `GLOBAL void jcopy_block_row(JBLOCKROW input_row, JBLOCKROW output_row, JDIMENSION num_blocks)`
- **Purpose:** Copies one row of DCT coefficient blocks (`JBLOCK` arrays)
- **Inputs:** Source/dest block rows, number of 8×8 blocks
- **Outputs/Return:** void; writes to `output_row`
- **Side effects:** Memory write
- **Calls:** `FMEMCOPY` or manual `JCOEF`-by-`JCOEF` loop
- **Notes:** Copies `num_blocks × DCTSIZE2` coefficients total

### jzero_far
- **Signature:** `GLOBAL void jzero_far(void FAR *target, size_t bytestozero)`
- **Purpose:** Zeroes a region of (potentially FAR-pointer) memory
- **Inputs:** Target pointer, byte count
- **Outputs/Return:** void; zeroes target buffer
- **Side effects:** Memory write
- **Calls:** `FMEMZERO` (resolves to `memset` or `_fmemset`) or byte loop fallback
- **Notes:** Handles large/virtual array allocations that may require FAR pointers on segmented 80x86

## Control Flow Notes
This file has no init/frame/shutdown lifecycle of its own. It is a passive utility module; its functions are called on-demand by both compression and decompression pipelines (e.g., during buffer management, coefficient processing, and array initialization). The two global tables are statically initialized at program load and read-only thereafter.

## External Dependencies
- `jinclude.h` — provides `MEMCOPY`, `MEMZERO`, `SIZEOF`, platform include dispatch
- `jpeglib.h` — provides `JSAMPARRAY`, `JBLOCKROW`, `JCOEFPTR`, `JDIMENSION`, `DCTSIZE2`, `JCOEF`, `JSAMPLE`, `FAR`, `GLOBAL`
- `jconfig.h` / `jmorecfg.h` (via `jpeglib.h`) — provide `NEED_FAR_POINTERS`, `USE_FMEM`, type sizes
- `memcpy` / `memset` / `_fmemcpy` / `_fmemset` — defined in system `<string.h>` or DOS far-memory library
