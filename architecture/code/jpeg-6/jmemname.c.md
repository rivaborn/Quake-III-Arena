# code/jpeg-6/jmemname.c

## File Purpose
Implements the system-dependent portion of the IJG JPEG memory manager for systems that require explicit temporary file naming. It provides memory allocation, memory availability reporting, and backing-store (temp file) management using named temporary files on disk.

## Core Responsibilities
- Allocate and free small and large memory objects via `malloc`/`free`
- Report available memory to the JPEG memory manager
- Generate unique temporary file names (via `mktemp` or manual polling)
- Open, read, write, and close backing-store temp files
- Initialize and terminate the memory subsystem

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `backing_store_info` | typedef struct (from `jmemsys.h`) | Holds temp file handle, name, and read/write/close function pointers for one backing-store object |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `next_file_num` | `static int` | file-static | Counter used to generate unique temp file name suffixes |

## Key Functions / Methods

### select_file_name
- **Signature:** `LOCAL void select_file_name(char *fname)`
- **Purpose:** Generates a unique temporary file name into `fname`.
- **Inputs:** `fname` — output buffer of at least `TEMP_NAME_LENGTH` bytes.
- **Outputs/Return:** Writes unique path string into `fname`; no return value.
- **Side effects:** Increments `next_file_num`. Without `NO_MKTEMP`: calls `mktemp()` which modifies `fname` in-place. With `NO_MKTEMP`: calls `fopen`/`fclose` in a loop until a non-existent filename is found.
- **Calls:** `sprintf`, `mktemp` (or `fopen`, `fclose`)
- **Notes:** Two compile-time variants controlled by `NO_MKTEMP`. The `NO_MKTEMP` path checks `errno == ENOENT` to distinguish "file not found" from other `fopen` failures.

### jpeg_get_small / jpeg_free_small
- **Signature:** `GLOBAL void* jpeg_get_small(j_common_ptr, size_t)` / `GLOBAL void jpeg_free_small(j_common_ptr, void*, size_t)`
- **Purpose:** Thin wrappers around `malloc`/`free` for small allocations.
- **Side effects:** Heap allocation/deallocation.

### jpeg_get_large / jpeg_free_large
- **Signature:** `GLOBAL void FAR* jpeg_get_large(j_common_ptr, size_t)` / `GLOBAL void jpeg_free_large(j_common_ptr, void FAR*, size_t)`
- **Purpose:** Same as small variants; no distinction on flat-memory systems.
- **Notes:** `FAR` keyword is vestigial for 80x86 segmented memory compatibility.

### jpeg_mem_available
- **Signature:** `GLOBAL long jpeg_mem_available(j_common_ptr cinfo, long min_bytes_needed, long max_bytes_needed, long already_allocated)`
- **Purpose:** Returns an estimate of remaining allocatable memory.
- **Outputs/Return:** `cinfo->mem->max_memory_to_use - already_allocated`
- **Notes:** Does not query OS; relies entirely on the `max_memory_to_use` limit set at init time.

### read_backing_store / write_backing_store
- **Signature:** `METHODDEF void read/write_backing_store(j_common_ptr, backing_store_ptr, void FAR*, long file_offset, long byte_count)`
- **Purpose:** Seek to `file_offset` in the temp file and read/write `byte_count` bytes.
- **Side effects:** File I/O on `info->temp_file`. Calls `ERREXIT` on seek or I/O failure.
- **Calls:** `fseek`, `JFREAD`/`JFWRITE`, `ERREXIT`

### close_backing_store
- **Signature:** `METHODDEF void close_backing_store(j_common_ptr, backing_store_ptr)`
- **Purpose:** Closes and deletes the temporary file.
- **Side effects:** `fclose` + `unlink` on `info->temp_file`/`info->temp_name`.
- **Notes:** Comment notes `remove()` as ANSI alternative to `unlink()`.

### jpeg_open_backing_store
- **Signature:** `GLOBAL void jpeg_open_backing_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- **Purpose:** Creates and opens a new temp file; installs read/write/close method pointers into `info`.
- **Side effects:** Calls `select_file_name`, `fopen`; populates `info->temp_file`, `info->temp_name`, and the three method pointers.
- **Calls:** `select_file_name`, `fopen`, `ERREXITS`, `TRACEMSS`

### jpeg_mem_init
- **Signature:** `GLOBAL long jpeg_mem_init(j_common_ptr cinfo)`
- **Purpose:** Resets `next_file_num` to 0 and returns `DEFAULT_MAX_MEM` (1 MB default).
- **Outputs/Return:** Default `max_memory_to_use` value.

### jpeg_mem_term
- **Signature:** `GLOBAL void jpeg_mem_term(j_common_ptr cinfo)`
- **Purpose:** No-op cleanup stub.

## Control Flow Notes
This file is not part of the game's frame loop. It is invoked during JPEG decode/encode operations (e.g., texture loading in `tr_image.c`). `jpeg_mem_init` is called at JPEG context creation; backing-store functions are called only if in-memory buffers are insufficient. `jpeg_mem_term` is called at JPEG context destruction.

## External Dependencies
- `jinclude.h` — platform stdio/string includes, `JFREAD`/`JFWRITE` macros
- `jpeglib.h` — `j_common_ptr`, `jpeg_memory_mgr`, `ERREXIT`, `TRACEMSS`
- `jmemsys.h` — `backing_store_ptr`, `backing_store_info`, function signatures
- `<errno.h>` — `ENOENT` (conditional, `NO_MKTEMP` path only)
- `malloc`, `free` — defined in `<stdlib.h>` or declared extern
- `mktemp`, `unlink`, `fopen`, `fclose`, `fseek` — defined in system libc
