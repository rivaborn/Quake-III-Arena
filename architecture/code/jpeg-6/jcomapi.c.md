# code/jpeg-6/jcomapi.c

## File Purpose
Provides the shared application interface routines for the IJG JPEG library that are common to both compression and decompression paths. It implements object lifecycle management (abort and destroy) and convenience allocators for quantization and Huffman tables.

## Core Responsibilities
- Abort an in-progress JPEG operation without destroying the object, resetting it for reuse
- Fully destroy a JPEG object and release all associated memory
- Allocate and zero-initialize quantization table (`JQUANT_TBL`) instances
- Allocate and zero-initialize Huffman table (`JHUFF_TBL`) instances

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `j_common_ptr` | typedef (pointer to struct) | Generic pointer to either a compress or decompress master struct; used so these routines work on both |
| `JQUANT_TBL` | struct | DCT coefficient quantization table; holds 64 quantization step values plus a `sent_table` flag |
| `JHUFF_TBL` | struct | Huffman coding table; holds bit-length counts and symbol values plus a `sent_table` flag |

## Global / File-Static State
None.

## Key Functions / Methods

### jpeg_abort
- **Signature:** `GLOBAL void jpeg_abort(j_common_ptr cinfo)`
- **Purpose:** Cleans up all non-permanent memory pools, closing any virtual-array temp files, then resets `global_state` so the object can be reused for another image.
- **Inputs:** `cinfo` — pointer to either a compress or decompress struct.
- **Outputs/Return:** void
- **Side effects:** Calls `cinfo->mem->free_pool` for every pool above `JPOOL_PERMANENT` (in reverse order). Writes `cinfo->global_state` to either `DSTATE_START` or `CSTATE_START`.
- **Calls:** `(*cinfo->mem->free_pool)(cinfo, pool)`
- **Notes:** Does **not** free the permanent pool; the object remains valid and re-enterable. Reverse pool order is a fragmentation hint for weak allocators.

### jpeg_destroy
- **Signature:** `GLOBAL void jpeg_destroy(j_common_ptr cinfo)`
- **Purpose:** Releases all memory and resources owned by the JPEG object via the memory manager's `self_destruct` method, then marks the object as destroyed.
- **Inputs:** `cinfo` — pointer to either a compress or decompress struct.
- **Outputs/Return:** void
- **Side effects:** Calls `(*cinfo->mem->self_destruct)(cinfo)` if `mem != NULL`. Sets `cinfo->mem = NULL` and `cinfo->global_state = 0`.
- **Calls:** `(*cinfo->mem->self_destruct)(cinfo)`
- **Notes:** Safe to call twice (guards with `mem != NULL` check). The `jpeg_compress_struct`/`jpeg_decompress_struct` itself and the error manager are **not** freed — those are the caller's responsibility.

### jpeg_alloc_quant_table
- **Signature:** `GLOBAL JQUANT_TBL *jpeg_alloc_quant_table(j_common_ptr cinfo)`
- **Purpose:** Allocates a `JQUANT_TBL` from the permanent pool and initializes `sent_table` to `FALSE`.
- **Inputs:** `cinfo`
- **Outputs/Return:** Pointer to a zeroed `JQUANT_TBL` (never NULL; exits on failure via error handler).
- **Side effects:** Allocates from `JPOOL_PERMANENT` via `cinfo->mem->alloc_small`.
- **Calls:** `(*cinfo->mem->alloc_small)(cinfo, JPOOL_PERMANENT, SIZEOF(JQUANT_TBL))`

### jpeg_alloc_huff_table
- **Signature:** `GLOBAL JHUFF_TBL *jpeg_alloc_huff_table(j_common_ptr cinfo)`
- **Purpose:** Allocates a `JHUFF_TBL` from the permanent pool and initializes `sent_table` to `FALSE`.
- **Inputs:** `cinfo`
- **Outputs/Return:** Pointer to a zeroed `JHUFF_TBL` (never NULL).
- **Side effects:** Allocates from `JPOOL_PERMANENT`.
- **Calls:** `(*cinfo->mem->alloc_small)(cinfo, JPOOL_PERMANENT, SIZEOF(JHUFF_TBL))`

## Control Flow Notes
This file has no frame/update/render involvement. Its functions are called during the **setup** and **teardown** phases of a JPEG operation:
- `jpeg_alloc_quant_table` / `jpeg_alloc_huff_table` — called during parameter setup before compression/decompression starts.
- `jpeg_abort` — called mid-operation to cancel and reset.
- `jpeg_destroy` — called at end of lifetime; corresponds to object shutdown.

## External Dependencies
- `jinclude.h` — platform portability macros (`SIZEOF`, `MEMZERO`, system headers)
- `jpeglib.h` — defines `j_common_ptr`, `JQUANT_TBL`, `JHUFF_TBL`, `jpeg_memory_mgr`, pool constants (`JPOOL_PERMANENT`, `JPOOL_NUMPOOLS`), state constants (`DSTATE_START`, `CSTATE_START`)
- `jpeg_memory_mgr::free_pool`, `::self_destruct`, `::alloc_small` — defined elsewhere (implemented in `jmemmgr.c`)
