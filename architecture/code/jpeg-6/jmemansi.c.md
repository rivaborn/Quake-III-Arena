# code/jpeg-6/jmemansi.c

## File Purpose
Provides the ANSI-standard, system-dependent memory management backend for the IJG JPEG library. It implements allocation via standard `malloc`/`free` and temporary file backing store via POSIX `tmpfile()` for overflow when available memory is insufficient.

## Core Responsibilities
- Allocate and free "small" heap objects via `malloc`/`free`
- Allocate and free "large" heap objects (same mechanism on flat-memory systems)
- Report available memory to the JPEG memory manager
- Create, read, write, and close temporary backing-store files using `tmpfile()`
- Provide memory subsystem init/term hooks (trivial in this implementation)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `backing_store_info` | struct (typedef, defined in `jmemsys.h`) | Holds `FILE* temp_file`, method pointers for read/write/close of a temp backing store |
| `backing_store_ptr` | typedef (pointer) | Pointer to a `backing_store_info`; passed to backing store routines |
| `j_common_ptr` | typedef (pointer) | Pointer to `jpeg_common_struct`; used to access error manager and memory limits |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `DEFAULT_MAX_MEM` | macro constant (`1000000L`) | file (compile-time) | Default ceiling for `max_memory_to_use` returned by `jpeg_mem_init` |

## Key Functions / Methods

### jpeg_get_small
- **Signature:** `GLOBAL void * jpeg_get_small(j_common_ptr cinfo, size_t sizeofobject)`
- **Purpose:** Allocate a small heap block.
- **Inputs:** `cinfo` (unused), `sizeofobject` — requested byte count.
- **Outputs/Return:** Pointer to allocated memory; NULL on failure (caller must handle).
- **Side effects:** Heap allocation via `malloc`.
- **Calls:** `malloc`
- **Notes:** `cinfo` not consulted; failure returns NULL (upper layer `jmemmgr.c` calls `ERREXIT` on NULL).

### jpeg_free_small
- **Signature:** `GLOBAL void jpeg_free_small(j_common_ptr cinfo, void *object, size_t sizeofobject)`
- **Purpose:** Free a previously small-allocated block.
- **Inputs:** `object` — pointer to free; size ignored.
- **Outputs/Return:** void.
- **Side effects:** Heap deallocation via `free`.
- **Calls:** `free`

### jpeg_get_large / jpeg_free_large
- Identical in behavior to `jpeg_get_small`/`jpeg_free_small` but use `FAR` pointers for 80x86 large-model compatibility. On flat 32-bit systems these are functionally the same.

### jpeg_mem_available
- **Signature:** `GLOBAL long jpeg_mem_available(j_common_ptr cinfo, long min_bytes_needed, long max_bytes_needed, long already_allocated)`
- **Purpose:** Reports how many more bytes can be allocated before backing store must be used.
- **Inputs:** `already_allocated` — bytes already in use; `cinfo->mem->max_memory_to_use` — configured ceiling.
- **Outputs/Return:** `max_memory_to_use - already_allocated` (may be negative if exceeded).
- **Side effects:** None.
- **Calls:** Nothing.
- **Notes:** `min_bytes_needed` and `max_bytes_needed` are ignored; this is a conservative estimate delegated to user configuration.

### jpeg_open_backing_store
- **Signature:** `GLOBAL void jpeg_open_backing_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- **Purpose:** Creates a temporary file and wires up read/write/close method pointers on `info`.
- **Inputs:** `info` — backing store record to initialize; `total_bytes_needed` — ignored.
- **Outputs/Return:** void; populates `info->temp_file` and method pointers.
- **Side effects:** Opens a temp file via `tmpfile()`; calls `ERREXITS` and aborts on failure.
- **Calls:** `tmpfile`, `ERREXITS`
- **Notes:** Temp file is anonymous (no name visible); deleted automatically on `fclose`.

### read_backing_store
- **Signature:** `METHODDEF void read_backing_store(j_common_ptr cinfo, backing_store_ptr info, void FAR *buffer_address, long file_offset, long byte_count)`
- **Purpose:** Seeks to `file_offset` in the temp file and reads `byte_count` bytes into `buffer_address`.
- **Side effects:** File I/O; calls `ERREXIT` on seek or read failure.
- **Calls:** `fseek`, `JFREAD`, `ERREXIT`

### write_backing_store
- **Signature:** `METHODDEF void write_backing_store(j_common_ptr cinfo, backing_store_ptr info, void FAR *buffer_address, long file_offset, long byte_count)`
- **Purpose:** Seeks and writes `byte_count` bytes from `buffer_address` to the temp file.
- **Side effects:** File I/O; calls `ERREXIT` on seek or write failure.
- **Calls:** `fseek`, `JFWRITE`, `ERREXIT`

### close_backing_store
- **Signature:** `METHODDEF void close_backing_store(j_common_ptr cinfo, backing_store_ptr info)`
- **Purpose:** Closes the temporary backing file.
- **Side effects:** `fclose` on `info->temp_file`; OS deletes the file automatically.
- **Calls:** `fclose`

### jpeg_mem_init
- **Signature:** `GLOBAL long jpeg_mem_init(j_common_ptr cinfo)`
- **Purpose:** Returns the default maximum memory budget.
- **Outputs/Return:** `DEFAULT_MAX_MEM` (1,000,000 bytes).
- **Notes:** No initialization work performed; return value is stored in `cinfo->mem->max_memory_to_use` by `jmemmgr.c`.

### jpeg_mem_term
- **Signature:** `GLOBAL void jpeg_mem_term(j_common_ptr cinfo)`
- **Purpose:** No-op cleanup hook.

## Control Flow Notes
This file is not called directly by the engine frame loop. It is invoked during JPEG decode/encode operations (image loading) initiated elsewhere. `jmemmgr.c` calls `jpeg_mem_init` at JPEG object creation, `jpeg_mem_available` before allocating virtual arrays, `jpeg_open_backing_store` only when memory is insufficient, and `jpeg_mem_term` at object destruction.

## External Dependencies
- `jinclude.h` — platform includes, `JFREAD`/`JFWRITE` macros, `SIZEOF`
- `jpeglib.h` — `j_common_ptr`, `jpeg_common_struct`, `jpeg_memory_mgr`
- `jmemsys.h` — `backing_store_info`, `backing_store_ptr`, function prototypes
- `malloc`, `free` — C standard library heap (ANSI `<stdlib.h>`)
- `tmpfile`, `fseek`, `fclose` — C standard I/O (`<stdio.h>`)
- `ERREXIT`, `ERREXITS` — defined elsewhere in the JPEG library (`jerror.h` / `jmemmgr.c`); perform error exit via `cinfo->err->error_exit`
