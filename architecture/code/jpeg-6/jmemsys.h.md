# code/jpeg-6/jmemsys.h

## File Purpose
Defines the interface between the system-independent JPEG memory manager (`jmemmgr.c`) and its system-dependent backend implementations. It declares the contract that any platform-specific memory manager must fulfill, covering small/large heap allocation, available-memory querying, and backing-store (temp file/XMS/EMS) management.

## Core Responsibilities
- Declare small-heap allocation/free functions (`jpeg_get_small`, `jpeg_free_small`)
- Declare large-heap allocation/free functions (`jpeg_get_large`, `jpeg_free_large`)
- Declare available-memory query (`jpeg_mem_available`)
- Define the `backing_store_info` struct with vtable-style method pointers for temp-file I/O
- Declare backing-store lifecycle functions (`jpeg_open_backing_store`, `jpeg_mem_init`, `jpeg_mem_term`)
- Provide short-name aliases for linkers with limited symbol-length support (`NEED_SHORT_EXTERNAL_NAMES`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `backing_store_info` | struct | Holds state and method pointers for a single backing-store object (temp file or XMS/EMS block) |
| `backing_store_ptr` | typedef (pointer) | Pointer alias for `backing_store_info` |
| `handle_union` | union (DOS-only) | Overlaps file handle, XMS handle, and EMS handle for MS-DOS backing-store variants |
| `XMSH` | typedef (`unsigned short`) | Opaque handle type for extended memory (XMS), MS-DOS only |
| `EMSH` | typedef (`unsigned short`) | Opaque handle type for expanded memory (EMS), MS-DOS only |

## Global / File-Static State

None.

## Key Functions / Methods

### jpeg_get_small
- **Signature:** `void * jpeg_get_small(j_common_ptr cinfo, size_t sizeofobject)`
- **Purpose:** Allocate a small chunk of memory (near heap on 80x86 small-model).
- **Inputs:** `cinfo` — JPEG common state; `sizeofobject` — bytes to allocate.
- **Outputs/Return:** Pointer to allocated memory, or `NULL` on failure.
- **Side effects:** System heap allocation.
- **Calls:** Defined elsewhere (system-dependent implementation).
- **Notes:** Semantically equivalent to `malloc`; must return `NULL` on failure.

### jpeg_free_small
- **Signature:** `void jpeg_free_small(j_common_ptr cinfo, void *object, size_t sizeofobject)`
- **Purpose:** Release a previously allocated small chunk.
- **Inputs:** `cinfo`, pointer to object, original size.
- **Outputs/Return:** void.
- **Side effects:** Frees heap memory.
- **Notes:** Size passed explicitly for systems where `free` requires it.

### jpeg_get_large / jpeg_free_large
- **Purpose:** Same contract as small variants but for large allocations; use `FAR` pointers on 80x86 segmented models.
- **Notes:** Allocation must not exceed `MAX_ALLOC_CHUNK` (default 1,000,000,000 bytes; overridable in `jconfig.h`).

### jpeg_mem_available
- **Signature:** `long jpeg_mem_available(j_common_ptr cinfo, long min_bytes_needed, long max_bytes_needed, long already_allocated)`
- **Purpose:** Report how many bytes are still usable for large allocations before backing-store is required.
- **Inputs:** Min/max bytes needed for operation; bytes already allocated.
- **Outputs/Return:** Estimated available bytes (may safely underestimate; must not overestimate).
- **Side effects:** None; query only.
- **Notes:** A 5% slop deduction from true free memory is recommended to avoid overestimation failures.

### jpeg_open_backing_store
- **Signature:** `void jpeg_open_backing_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- **Purpose:** Open a backing-store object and populate its `read_backing_store`, `write_backing_store`, and `close_backing_store` function pointers.
- **Inputs:** `cinfo`, uninitialized `backing_store_info`, max bytes the store must hold.
- **Outputs/Return:** void; fills in `info` in-place.
- **Side effects:** Opens a temp file (or allocates XMS/EMS block); may `error_exit` if size exceeded.

### jpeg_mem_init / jpeg_mem_term
- **Signature:** `long jpeg_mem_init(j_common_ptr cinfo)` / `void jpeg_mem_term(j_common_ptr cinfo)`
- **Purpose:** One-time init/cleanup for the memory subsystem. `jpeg_mem_init` runs before any allocation; `jpeg_mem_term` runs after all memory is freed and all backing stores are closed.
- **Outputs/Return:** `jpeg_mem_init` returns a recommended default for `max_memory_to_use`.
- **Notes:** At `jpeg_mem_init` time, only the error manager pointer in `cinfo` is valid.

## Control Flow Notes
This header is included only by `jmemmgr.c` (system-independent) and the chosen system-dependent implementation (e.g., `jmemansi.c`, `jmemdos.c`, `jmemnobs.c`). It is part of JPEG library initialization: `jpeg_mem_init` is called early in `jpeg_CreateCompress`/`jpeg_CreateDecompress`; backing-store objects are opened lazily if in-memory buffers prove insufficient during a compression or decompression pass; `jpeg_mem_term` is called during `jpeg_destroy`.

## External Dependencies
- `jpeglib.h` / `jpegint.h` — for `j_common_ptr`, `JMETHOD`, `JPP`, `FAR`, `EXTERN` macros
- `jconfig.h` — supplies `USE_MSDOS_MEMMGR`, `NEED_SHORT_EXTERNAL_NAMES`, `MAX_ALLOC_CHUNK` overrides
- `<stdio.h>` — `FILE *` used in the non-DOS `backing_store_info` branch
- All declared functions are **defined elsewhere** in one of: `jmemansi.c`, `jmemnobs.c`, `jmemdos.c`, `jmemname.c` (selected at build time)
