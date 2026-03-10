# code/jpeg-6/jmemdos.c

## File Purpose
MS-DOS-specific implementation of the IJG JPEG memory manager's system-dependent layer. Provides heap allocation and three types of backing store (DOS files, XMS extended memory, EMS expanded memory) for spilling JPEG working buffers when RAM is insufficient.

## Core Responsibilities
- Allocate and free small (near heap) and large (far heap) memory blocks
- Report available memory to the JPEG memory manager
- Select and generate unique temporary file names using the `TMP`/`TEMP` environment variables
- Open, read, write, and close DOS-file-based backing store via direct DOS calls (assembly stubs)
- Open, read, write, and close XMS (extended memory, V2.0) backing store via the XMS driver
- Open, read, write, and close EMS (expanded memory, LIM/EMS 4.0) backing store via the EMS driver
- Initialize and terminate the memory subsystem (`jpeg_mem_init`, `jpeg_mem_term`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `XMScontext` | struct | Register state passed to/from the XMS driver via `jxms_calldriver` |
| `XMSspec` | struct | XMS EMB move specification (source handle/offset, dest handle/offset, length) |
| `EMSspec` | union | EMS move specification with misaligned fields accessed via byte-offset macros |
| `XMSDRIVER` | typedef (`void far *`) | Pointer to the XMS driver entry point |
| `XMSPTR` | union | Either a 32-bit flat offset or a real-mode far pointer, used in `XMSspec` |
| `EMSPTR` | typedef (`void far *`) | Far pointer used in EMS move specs |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `next_file_num` | `int` | static (file) | Counter for generating unique temporary file name suffixes (`JPG001.TMP`, etc.) |
| `xms_driver` | `XMSDRIVER` | static (file, XMS block) | Cached far-pointer to the XMS driver; obtained once per `open_xms_store` call |

## Key Functions / Methods

### select_file_name
- **Signature:** `LOCAL void select_file_name(char *fname)`
- **Purpose:** Generates a unique temporary file path by combining the `TMP`/`TEMP` env-var directory with an incrementing counter suffix.
- **Inputs:** `fname` — output buffer (caller-allocated, ≥ `TEMP_NAME_LENGTH` bytes)
- **Outputs/Return:** Writes the path into `fname`; no return value.
- **Side effects:** Increments `next_file_num`; calls `fopen`/`fclose` to probe for name collisions; reads environment variables.
- **Calls:** `getenv`, `sprintf`, `fopen`, `fclose`
- **Notes:** Loops until a name that does not yet exist is found; uses `\` as path separator (DOS convention).

### jpeg_get_small / jpeg_free_small
- **Signature:** `GLOBAL void* jpeg_get_small(j_common_ptr, size_t)` / `GLOBAL void jpeg_free_small(j_common_ptr, void*, size_t)`
- **Purpose:** Near-heap allocator/deallocator wrapping standard `malloc`/`free`.
- **Side effects:** None beyond heap modification.

### jpeg_get_large / jpeg_free_large
- **Signature:** `GLOBAL void FAR* jpeg_get_large(j_common_ptr, size_t)` / `GLOBAL void jpeg_free_large(j_common_ptr, void FAR*, size_t)`
- **Purpose:** Far-heap allocator/deallocator using `far_malloc`/`far_free` (mapped to `farmalloc`/`_fmalloc` or plain `malloc` depending on memory model).
- **Side effects:** None beyond heap modification.

### jpeg_mem_available
- **Signature:** `GLOBAL long jpeg_mem_available(j_common_ptr cinfo, long min, long max, long already_allocated)`
- **Purpose:** Returns estimated remaining allocatable bytes (`max_memory_to_use - already_allocated`).
- **Notes:** No dynamic probing; relies entirely on the compile-time `DEFAULT_MAX_MEM` (300 000 bytes) as the ceiling.

### open_xms_store
- **Signature:** `LOCAL boolean open_xms_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- **Purpose:** Queries the XMS driver, checks version ≥ 2.00, allocates an Extended Memory Block, and wires `read_xms_store`/`write_xms_store`/`close_xms_store` into `info`.
- **Side effects:** Sets global `xms_driver`; allocates XMS memory.
- **Calls:** `jxms_getdriver`, `jxms_calldriver`

### read_xms_store / write_xms_store
- **Signature:** `METHODDEF void read/write_xms_store(j_common_ptr, backing_store_ptr, void FAR*, long, long)`
- **Purpose:** Transfer data between a FAR buffer and the XMS block using the EMB-move call (AX=0x0B00).
- **Notes:** XMS requires even byte counts; odd-byte transfers recurse with a 2-byte `endbuffer` to handle the final byte.
- **Side effects:** Calls `ERREXIT` on driver failure.

### open_ems_store
- **Signature:** `LOCAL boolean open_ems_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- **Purpose:** Verifies EMS availability, checks version ≥ 4.0, allocates EMS pages, and wires the EMS read/write/close methods.
- **Calls:** `jems_available`, `jems_calldriver`

### read_ems_store / write_ems_store
- **Signature:** `METHODDEF void read/write_ems_store(j_common_ptr, backing_store_ptr, void FAR*, long, long)`
- **Purpose:** Transfer data between a FAR buffer and EMS pages using the EMS move-memory-region call (AX=0x5700).
- **Notes:** Uses misaligned-field macros (`SRC_PAGE`, `DST_PAGE`, etc.) because the EMS spec structure crosses alignment boundaries.

### jpeg_open_backing_store
- **Signature:** `GLOBAL void jpeg_open_backing_store(j_common_ptr cinfo, backing_store_ptr info, long total_bytes_needed)`
- **Purpose:** Entry point called by `jmemmgr.c`; tries XMS → EMS → DOS file in order, `ERREXIT`s if all fail.
- **Calls:** `open_xms_store`, `open_ems_store`, `open_file_store`

### jpeg_mem_init / jpeg_mem_term
- **Signature:** `GLOBAL long jpeg_mem_init(j_common_ptr)` / `GLOBAL void jpeg_mem_term(j_common_ptr)`
- **Purpose:** `jpeg_mem_init` resets `next_file_num` and returns `DEFAULT_MAX_MEM`. `jpeg_mem_term` optionally calls `_fheapmin()` (MSVC 6 workaround for large-block reclaim failures).
- **Side effects:** `jpeg_mem_init` modifies `next_file_num`.

## Control Flow Notes
This file is not on the frame/render path. It is called during JPEG compression/decompression **initialization** (`jpeg_mem_init`) and **shutdown** (`jpeg_mem_term`), and on-demand during decoding when virtual array backing store is required (`jpeg_open_backing_store` → read/write methods). The entry points are called exclusively by `jmemmgr.c`.

## External Dependencies
- `jinclude.h`, `jpeglib.h`, `jmemsys.h` — IJG JPEG library internals
- `<alloc.h>` (Turbo C) or `<malloc.h>` (MSVC) — far-heap routines
- `<stdlib.h>` — `malloc`, `free`, `getenv`
- Assembly stubs in `jmemdosa.asm` (defined elsewhere): `jdos_open`, `jdos_close`, `jdos_seek`, `jdos_read`, `jdos_write`, `jxms_getdriver`, `jxms_calldriver`, `jems_available`, `jems_calldriver`
- `ERREXIT`, `ERREXITS`, `TRACEMSS`, `TRACEMS1` — error/trace macros defined in `jerror.h` (via `jpeglib.h`)
