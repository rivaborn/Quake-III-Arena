# code/jpeg-6/jmemmgr.c

## File Purpose
Implements the system-independent JPEG memory manager for the IJG JPEG library. It provides pool-based allocation (small and large objects), 2-D array allocation for image samples and DCT coefficient blocks, and virtual array management with optional disk-backed overflow storage.

## Core Responsibilities
- Pool-based allocation and lifetime management of "small" and "large" memory objects across `JPOOL_PERMANENT` and `JPOOL_IMAGE` lifetimes
- Allocation of 2-D sample arrays (`JSAMPARRAY`) and coefficient-block arrays (`JBLOCKARRAY`) with chunked large-object backing
- Registration and deferred realization of virtual (potentially disk-backed) sample and block arrays
- Swapping virtual array strips between in-memory buffers and backing store on demand
- Tracking total allocated space and enforcing `max_memory_to_use` policy
- Teardown: freeing all pools (including closing backing-store files) and destroying the manager itself

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `small_pool_hdr` | union (struct + ALIGN_TYPE dummy) | Header for a small memory pool block; tracks next pointer, bytes used/left |
| `large_pool_hdr` | union (struct + FAR ALIGN_TYPE dummy) | Header for a large memory pool block; same fields as small |
| `my_memory_mgr` | struct (typedef) | Full memory manager state: extends public `jpeg_memory_mgr` with pool lists, virtual array lists, and total allocation counter |
| `jvirt_sarray_control` | struct | Control block for a virtual sample array; tracks buffer, dimensions, dirty flag, backing store info |
| `jvirt_barray_control` | struct | Control block for a virtual coefficient-block array; same pattern as sarray |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `first_pool_slop` | `static const size_t[JPOOL_NUMPOOLS]` | static | Initial over-allocation slop per pool class (1600 bytes permanent, 16000 bytes image) |
| `extra_pool_slop` | `static const size_t[JPOOL_NUMPOOLS]` | static | Additional slop for subsequent pools (0 permanent, 5000 image) |

## Key Functions / Methods

### `alloc_small`
- Signature: `METHODDEF void * alloc_small(j_common_ptr cinfo, int pool_id, size_t sizeofobject)`
- Purpose: Allocates a small object from a pool. Reuses space in existing pools; creates a new pool with slop if needed.
- Inputs: `cinfo` (JPEG context), `pool_id`, `sizeofobject`
- Outputs/Return: `void *` pointer to allocated memory; never NULL
- Side effects: May call `jpeg_get_small`, updates `total_space_allocated`, modifies `small_list[pool_id]`
- Calls: `jpeg_get_small`, `out_of_memory`, `ERREXIT1`
- Notes: Size is rounded up to `ALIGN_TYPE` alignment. Slop is halved on allocation failure until below `MIN_SLOP`.

### `alloc_large`
- Signature: `METHODDEF void FAR * alloc_large(j_common_ptr cinfo, int pool_id, size_t sizeofobject)`
- Purpose: Allocates a large object; each request becomes its own pool node (no sub-allocation sharing).
- Inputs: `cinfo`, `pool_id`, `sizeofobject`
- Outputs/Return: `void FAR *`; never NULL
- Side effects: Calls `jpeg_get_large`, updates `total_space_allocated`, prepends to `large_list[pool_id]`
- Calls: `jpeg_get_large`, `out_of_memory`, `ERREXIT1`

### `alloc_sarray`
- Signature: `METHODDEF JSAMPARRAY alloc_sarray(j_common_ptr cinfo, int pool_id, JDIMENSION samplesperrow, JDIMENSION numrows)`
- Purpose: Allocates a 2-D sample array; row pointers in small pool, row data in large pool chunks up to `MAX_ALLOC_CHUNK`.
- Inputs: `cinfo`, `pool_id`, `samplesperrow`, `numrows`
- Outputs/Return: `JSAMPARRAY`
- Side effects: Sets `mem->last_rowsperchunk`; calls `alloc_small` and `alloc_large`
- Notes: `last_rowsperchunk` is read by `realize_virt_arrays`.

### `alloc_barray`
- Signature: `METHODDEF JBLOCKARRAY alloc_barray(j_common_ptr cinfo, int pool_id, JDIMENSION blocksperrow, JDIMENSION numrows)`
- Purpose: Same as `alloc_sarray` but for `JBLOCK` coefficient arrays.
- Side effects: Same pattern; sets `mem->last_rowsperchunk`

### `request_virt_sarray` / `request_virt_barray`
- Purpose: Register a virtual array need before memory is committed; stores control block in small pool. Actual buffer allocation is deferred to `realize_virt_arrays`.
- Outputs: `jvirt_sarray_ptr` / `jvirt_barray_ptr`; prepended to `virt_sarray_list`/`virt_barray_list`

### `realize_virt_arrays`
- Signature: `METHODDEF void realize_virt_arrays(j_common_ptr cinfo)`
- Purpose: Allocates in-memory buffers for all unresolved virtual arrays. Queries `jpeg_mem_available` to decide how many rows fit in memory; opens backing store for overflow.
- Side effects: Calls `alloc_sarray`/`alloc_barray`, `jpeg_open_backing_store`; sets `rows_in_mem`, `rowsperchunk`, initializes `cur_start_row`/`first_undef_row`/`dirty`

### `access_virt_sarray` / `access_virt_barray`
- Purpose: Makes a strip of rows accessible, swapping in/out from backing store if the requested range is outside the current in-memory window.
- Inputs: `cinfo`, array pointer, `start_row`, `num_rows`, `writable`
- Outputs/Return: Pointer into the in-memory buffer at `start_row - cur_start_row`
- Side effects: May call `do_sarray_io`/`do_barray_io`; sets `dirty`, pre-zeros uninitialized rows via `jzero_far` if `pre_zero` is set
- Notes: Errors on out-of-bounds access or reading undefined data without pre-zero.

### `free_pool`
- Purpose: Releases all objects in a pool. For `JPOOL_IMAGE`, first closes all virtual array backing stores.
- Side effects: Calls `jpeg_free_large`, `jpeg_free_small`, decrements `total_space_allocated`; closes backing stores via function pointer `close_backing_store`

### `self_destruct`
- Purpose: Tears down all pools in reverse order, frees the manager control block itself, calls `jpeg_mem_term`.
- Side effects: Sets `cinfo->mem = NULL`

### `jinit_memory_mgr`
- Signature: `GLOBAL void jinit_memory_mgr(j_common_ptr cinfo)`
- Purpose: Entry point called during JPEG object creation. Validates alignment/chunk constants, calls `jpeg_mem_init`, allocates `my_memory_mgr`, wires all method pointers, optionally reads `JPEGMEM` environment variable to override memory limit.
- Side effects: Sets `cinfo->mem`; calls `jpeg_mem_init`; reads `getenv("JPEGMEM")`

## Control Flow Notes
`jinit_memory_mgr` is called during JPEG library initialization (before any compression/decompression). `realize_virt_arrays` is called once before scanning begins. `access_virt_sarray`/`access_virt_barray` are called per strip during encode/decode passes. `free_pool(JPOOL_IMAGE)` is called at end-of-image; `self_destruct` at destroy time.

## External Dependencies
- `jinclude.h`, `jpeglib.h`, `jmemsys.h`
- **Defined elsewhere:** `jpeg_get_small`, `jpeg_free_small`, `jpeg_get_large`, `jpeg_free_large`, `jpeg_mem_available`, `jpeg_open_backing_store`, `jpeg_mem_init`, `jpeg_mem_term` (system-dependent, e.g., `jmemnobs.c` or `jmemansi.c`), `jzero_far`, `ERREXIT`/`ERREXIT1` macros (error handler)
