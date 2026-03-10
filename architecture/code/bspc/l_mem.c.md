# code/bspc/l_mem.c

## File Purpose
Provides memory allocation, tracking, and management for the BSPC (BSP Compiler) tool. Implements both a lightweight release mode and a debug mode (`MEMDEBUG`) with full block introspection, plus compatibility shims for Quake 3's `Hunk_*` and `Z_*` memory APIs.

## Core Responsibilities
- Allocate (`GetMemory`) and zero-initialize (`GetClearedMemory`) heap memory via `malloc`
- Track total allocated memory size via a global counter (`allocedmemory`) in release mode
- In debug mode, maintain a doubly-linked list of tagged `memoryblock_t` headers for leak detection and validation
- Provide `FreeMemory` with block validation (magic ID, pointer self-consistency) in debug mode
- Implement a simple linked-list hunk allocator (`Hunk_Alloc`, `Hunk_ClearHigh`) over `GetClearedMemory`
- Bridge Quake 3 engine `Z_Malloc`/`Z_Free` calls to the local allocator
- Log memory size summaries (`PrintMemorySize`, `PrintMemoryLabels`, `PrintUsedMemorySize`)

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `memoryblock_t` | struct | Debug-mode block header prepended to each allocation; carries magic ID, user pointer, size, and optional source label/file/line |
| `memhunk_t` | struct | Node in the hunk allocator's linked list; carries a pointer to the usable region and a `next` pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `allocedmemory` | `int` | global | Running total of allocated bytes (release mode only) |
| `totalmemorysize` | `int` | global (MEMDEBUG) | Total bytes across all live debug blocks |
| `numblocks` | `int` | global (MEMDEBUG) | Count of live debug allocations |
| `memory` | `memoryblock_t *` | global (MEMDEBUG) | Head of the doubly-linked block list |
| `memhunk_high` | `memhunk_t *` | global | Head of the high-hunk linked list |
| `memhunk_low` | `memhunk_t *` | global | Head of the low-hunk list (unused; always NULL) |
| `memhunk_high_size` | `int` | global | Remaining hunk budget (starts at 16 MB, decremented on each `Hunk_Alloc`) |
| `memhunk_low_size` | `int` | global | Low-hunk budget (unused) |

## Key Functions / Methods

### PrintMemorySize
- **Signature:** `void PrintMemorySize(unsigned long size)`
- **Purpose:** Formats a byte count into MB / KB / bytes and prints via `Log_Print`.
- **Inputs:** `size` — byte count to format.
- **Outputs/Return:** None.
- **Side effects:** Writes to the log.
- **Calls:** `Log_Print`
- **Notes:** Always active regardless of `MEMDEBUG`.

### GetMemory / GetMemoryDebug
- **Signature:** `void *GetMemory(unsigned long size)` / `void *GetMemoryDebug(unsigned long size, char *label, char *file, int line)`
- **Purpose:** Allocates `size` bytes. In debug mode prepends a `memoryblock_t` header with magic ID and source metadata.
- **Inputs:** `size`; debug variant also takes label, filename, and line number.
- **Outputs/Return:** Pointer to usable memory.
- **Side effects:** Updates `allocedmemory` (release) or `totalmemorysize`/`numblocks` + links into `memory` list (debug). Calls `Error` on allocation failure.
- **Calls:** `malloc`, `LinkMemoryBlock` (debug), `Error`
- **Notes:** `MEM_ID = 0x12345678` used as block sentinel.

### GetClearedMemory / GetClearedMemoryDebug
- **Signature:** `void *GetClearedMemory(int size)` / debug variant
- **Purpose:** Allocates and zero-initializes `size` bytes.
- **Inputs:** `size`.
- **Outputs/Return:** Zeroed pointer.
- **Side effects:** Same as `GetMemory`; calls `memset`.
- **Calls:** `GetMemory` (or `GetMemoryDebug`), `memset`, `MemorySize`, `Error`

### FreeMemory
- **Signature:** `void FreeMemory(void *ptr)`
- **Purpose:** Frees memory; in debug mode validates block integrity before freeing.
- **Inputs:** `ptr` — pointer previously returned by `GetMemory`/`GetClearedMemory`.
- **Outputs/Return:** None.
- **Side effects:** Updates accounting globals; unlinks from `memory` list (debug).
- **Calls:** `BlockFromPointer` (debug), `UnlinkMemoryBlock` (debug), `free`, `MemorySize` (release)
- **Notes:** Release mode uses `_msize` (Windows) to account for actual block size; returns 0 on non-Windows.

### BlockFromPointer
- **Signature:** `memoryblock_t *BlockFromPointer(void *ptr, char *str)`
- **Purpose:** Walks back from user pointer to the prepended `memoryblock_t` and validates magic ID and self-pointer.
- **Inputs:** `ptr` — user data pointer; `str` — caller name for error messages.
- **Outputs/Return:** Valid `memoryblock_t *` or NULL (if ptr is NULL and `MEMDEBUG` is off).
- **Side effects:** Calls `Error` on invalid block.
- **Calls:** `Error`

### Hunk_Alloc
- **Signature:** `void *Hunk_Alloc(int size)`
- **Purpose:** Allocates from the high hunk; if `size == 0` returns remaining budget as a `(void *)` integer.
- **Inputs:** `size`.
- **Outputs/Return:** Pointer to zeroed region, or `(void *)memhunk_high_size` if size is 0.
- **Side effects:** Decrements `memhunk_high_size`; prepends to `memhunk_high` list.
- **Calls:** `GetClearedMemory`

### Hunk_ClearHigh
- **Signature:** `void Hunk_ClearHigh(void)`
- **Purpose:** Frees all high-hunk allocations and resets the budget to 16 MB.
- **Calls:** `FreeMemory`

### Z_Malloc / Z_Free
- **Signature:** `void *Z_Malloc(int size)` / `void Z_Free(void *ptr)`
- **Purpose:** Thin wrappers mapping Quake 3 zone-memory API to `GetClearedMemory`/`FreeMemory`.

### DumpMemory / PrintMemoryLabels / TotalAllocatedMemory
- Notes: Debug-only utilities; `DumpMemory` iterates `memory` freeing all blocks; `PrintMemoryLabels` dumps per-block source info via `Log_Write`; `TotalAllocatedMemory` returns the current total.

## Control Flow Notes
This file has no per-frame update role. It is a utility module used throughout BSPC's initialization and BSP processing pipeline. `Hunk_ClearHigh` would be called at tool shutdown or between map conversions. `DumpMemory` is a cleanup/shutdown call.

## External Dependencies
- **Includes:** `qbsp.h` (pulls in `malloc.h`, BSP types, and all bspc headers), `l_log.h`
- **External symbols used but not defined here:**
  - `Error` — fatal error handler (defined in `l_cmd.c`)
  - `Log_Print`, `Log_Write` — logging (defined in `l_log.c`)
  - `_msize` — Win32 CRT heap query (platform SDK)
  - `malloc`, `free`, `memset` — C standard library
