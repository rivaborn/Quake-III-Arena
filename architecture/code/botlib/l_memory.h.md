# code/botlib/l_memory.h

## File Purpose
Public interface for the botlib's internal memory management subsystem. It declares allocation/deallocation functions for both standard heap memory and hunk (engine-side) memory, with optional debug instrumentation via the `MEMDEBUG` preprocessor toggle.

## Core Responsibilities
- Declare heap allocation functions (`GetMemory`, `GetClearedMemory`)
- Declare hunk allocation functions (`GetHunkMemory`, `GetClearedHunkMemory`) or alias them to heap variants under `BSPC`
- Provide debug variants that capture allocation label, source file, and line number
- Macro-redirect allocation calls transparently when `MEMDEBUG` is defined
- Declare deallocation, introspection, and bulk-free utilities

## Key Types / Data Structures
None.

## Global / File-Static State
None.

## Key Functions / Methods

### GetMemory / GetMemoryDebug
- Signature: `void *GetMemory(unsigned long size)` / `void *GetMemoryDebug(unsigned long size, char *label, char *file, int line)`
- Purpose: Allocate a raw (uninitialised) block of heap memory for botlib use.
- Inputs: `size` — byte count; debug variant also takes a stringified label, filename, and line number.
- Outputs/Return: Pointer to allocated block; behavior on failure not inferable from this file.
- Side effects: Heap allocation; in debug mode, records metadata for leak tracking.
- Calls: Defined in `l_memory.c`.
- Notes: Under `MEMDEBUG`, the macro stringifies the `size` expression as the label.

### GetClearedMemory / GetClearedMemoryDebug
- Signature: `void *GetClearedMemory(unsigned long size)` / debug variant analogous to above.
- Purpose: Allocate a zero-initialised heap block (equivalent of `calloc`).
- Inputs/Outputs: Same as `GetMemory`.
- Side effects: Same as `GetMemory` plus zero-fill.
- Notes: Preferred when callers need guaranteed zeroed state (e.g., struct initialisation).

### GetHunkMemory / GetClearedHunkMemory
- Signature: `void *GetHunkMemory(unsigned long size)` / cleared variant.
- Purpose: Allocate from the engine hunk allocator rather than the generic heap.
- Inputs/Outputs: Same as heap variants.
- Side effects: Hunk-side allocation; under `BSPC` build these are aliased to `GetMemory`/`GetClearedMemory` via `#define`.
- Notes: Separation allows the botlib to route long-lived or level-scoped data through the engine's hunk while using the heap for shorter-lived allocations.

### FreeMemory
- Signature: `void FreeMemory(void *ptr)`
- Purpose: Release a block previously returned by any `Get*Memory` function.
- Side effects: Heap/hunk deallocation; in debug mode updates tracking state.

### DumpMemory
- Signature: `void DumpMemory(void)`
- Purpose: Free all memory blocks allocated through this subsystem in one pass.
- Side effects: Bulk deallocation; likely used at shutdown or level unload.

### Notes (minor utilities)
- `AvailableMemory()` — returns remaining allocatable memory (int).
- `PrintUsedMemorySize()` — logs total bytes in use; debug/diagnostic output.
- `PrintMemoryLabels()` — prints all live allocations with their labels; only meaningful with `MEMDEBUG`.
- `MemoryByteSize(void *ptr)` — returns the byte size of a live allocation.

## Control Flow Notes
This is a header-only declaration file. The implementations live in `l_memory.c`. The allocators are called throughout botlib initialisation, per-frame AI processing, and shutdown. `DumpMemory` maps to a shutdown path. The `BSPC` conditional collapses hunk/heap distinctions for the standalone BSP compiler tool, which has no engine hunk.

## External Dependencies
- No includes in this file itself.
- Implementations in `l_memory.c` depend on engine trap functions or `malloc`/`free` (not inferable here).
- `BSPC` and `MEMDEBUG` — compile-time defines set by the build system.
- `FreeMemory`, `GetMemory`, etc. are consumed by virtually all other botlib `.c` files.
