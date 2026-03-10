# code/botlib/l_memory.c

## File Purpose
Provides the botlib's memory allocation abstraction layer, delegating all actual allocations to the engine via the `botimport` function table. Supports two compile-time configurations: a full memory manager (`MEMORYMANEGER`) with block tracking, and a lightweight mode that only prepends a magic ID word.

## Core Responsibilities
- Wrap `botimport.GetMemory` and `botimport.HunkAlloc` with bookkeeping headers
- Validate pointers on free by checking magic IDs (`MEM_ID` / `HUNK_ID`)
- Track total allocated bytes, total botlib memory, and block count (manager mode only)
- Provide zeroed variants of both heap and hunk allocators
- Dump all live allocations and report memory usage (manager mode)
- Conditionally compile debug variants that record label, file, and line per allocation

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `memoryblock_t` | struct (MEMORYMANEGER only) | Per-allocation header storing magic ID, user pointer, size, linked-list links, and optional debug source info |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `allocatedmemory` | `int` | global | Running total of allocated bytes (payload + header) |
| `totalmemorysize` | `int` | global | Total botlib memory including `memoryblock_t` overhead |
| `numblocks` | `int` | global | Count of live allocation blocks |
| `memory` | `memoryblock_t *` | global (MEMORYMANEGER) | Head of doubly-linked list of all live blocks |

## Key Functions / Methods

### GetMemory
- Signature: `void *GetMemory(unsigned long size)` (or `GetMemoryDebug` with label/file/line under MEMDEBUG)
- Purpose: Allocate heap memory from the engine Zone, prepend a tracking header.
- Inputs: `size` — requested byte count
- Outputs/Return: Pointer to usable memory (past the header)
- Side effects: Increments `allocatedmemory`, `totalmemorysize`, `numblocks`; links block into `memory` list (manager mode)
- Calls: `botimport.GetMemory`, `LinkMemoryBlock`
- Notes: In lightweight mode, only stores `MEM_ID` word; no list maintenance.

### GetHunkMemory
- Signature: `void *GetHunkMemory(unsigned long size)`
- Purpose: Allocate from the engine hunk (permanent, non-freeable allocator); prepend header tagged `HUNK_ID`.
- Inputs: `size`
- Outputs/Return: Pointer past header
- Side effects: Same counters as `GetMemory`; links into `memory` list (manager mode)
- Calls: `botimport.HunkAlloc`, `LinkMemoryBlock`

### GetClearedMemory / GetClearedHunkMemory
- Call `GetMemory`/`GetHunkMemory` then zero the returned buffer with `Com_Memset`.

### FreeMemory
- Signature: `void FreeMemory(void *ptr)`
- Purpose: Validate and release a heap allocation. Hunk allocations (`HUNK_ID`) are intentionally not freed.
- Inputs: `ptr` — user pointer previously returned by `GetMemory`
- Outputs/Return: void
- Side effects: Decrements counters; unlinks block; calls `botimport.FreeMemory` on the raw block pointer
- Calls: `BlockFromPointer`, `UnlinkMemoryBlock`, `botimport.FreeMemory`
- Notes: Lightweight mode skips unlink; checks `*memid == MEM_ID` directly.

### BlockFromPointer *(MEMORYMANEGER only)*
- Signature: `memoryblock_t *BlockFromPointer(void *ptr, char *str)`
- Purpose: Recover and validate the `memoryblock_t` header from a user pointer.
- Inputs: `ptr`, diagnostic string `str`
- Outputs/Return: Validated `memoryblock_t *` or NULL on error
- Side effects: Calls `botimport.Print(PRT_FATAL, ...)` on invalid block
- Notes: Checks both ID magic and pointer self-reference.

### DumpMemory *(MEMORYMANEGER only)*
- Iterates the `memory` list calling `FreeMemory` on each block; resets counters to zero afterward.

### PrintMemoryLabels / PrintUsedMemorySize
- Diagnostic output; no-ops in lightweight mode. Manager mode logs each block via `Log_Write`.

### AvailableMemory
- Passthrough to `botimport.AvailableMemory()`.

## Control Flow Notes
This file has no frame/update role. It is used throughout botlib initialization and runtime whenever any botlib subsystem allocates or frees memory. `DumpMemory` is called during botlib shutdown to reclaim all allocations.

## External Dependencies
- `../game/q_shared.h` — `Com_Memset`, basic types
- `../game/botlib.h` — `botlib_import_t` definition (provides `GetMemory`, `FreeMemory`, `HunkAlloc`, `AvailableMemory`, `Print`)
- `l_log.h` — `Log_Write` (used by `PrintMemoryLabels`)
- `be_interface.h` — `botimport` extern (the live `botlib_import_t` instance)
- `botimport` — defined in `be_interface.c`; all actual memory operations delegate through it
