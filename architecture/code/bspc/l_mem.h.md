# code/bspc/l_mem.h

## File Purpose
Public interface header for the BSPC tool's custom memory management subsystem. It declares allocation and deallocation routines with an optional debug mode that captures source location metadata at the call site via macros.

## Core Responsibilities
- Declare `GetMemory` / `GetClearedMemory` for raw and zero-initialized heap allocation
- Provide debug-mode macro overrides that inject label, file, and line information into every allocation call
- Declare `FreeMemory` as the single deallocation entry point
- Expose utility queries: per-block size (`MemorySize`), human-readable size printing (`PrintMemorySize`), and total allocated byte count (`TotalAllocatedMemory`)

## Key Types / Data Structures
None.

## Global / File-Static State
None declared in this header.

## Key Functions / Methods

### GetMemory / GetClearedMemory (non-debug)
- **Signature:** `void *GetMemory(unsigned long size)` / `void *GetClearedMemory(int size)`
- **Purpose:** Allocate a raw or zero-initialized block of `size` bytes from the custom heap.
- **Inputs:** `size` — byte count requested.
- **Outputs/Return:** Pointer to allocated memory block; error behavior not inferable from header.
- **Side effects:** Modifies internal heap state (defined elsewhere).
- **Calls:** Not inferable from this file.
- **Notes:** `GetClearedMemory` takes `int` while `GetMemory` takes `unsigned long` — a minor signature inconsistency.

### GetMemoryDebug / GetClearedMemoryDebug (debug mode only)
- **Signature:** `void *GetMemoryDebug(unsigned long size, char *label, char *file, int line)` / `void *GetClearedMemoryDebug(unsigned long size, char *label, char *file, int line)`
- **Purpose:** Debug variants that record a string label (stringified size expression), source file, and line number alongside the allocation for leak/corruption diagnostics.
- **Inputs:** `size`, `label` (stringified via `#size`), `file` (`__FILE__`), `line` (`__LINE__`).
- **Outputs/Return:** Pointer to allocated block.
- **Side effects:** Stores metadata with each allocation; feeds `PrintMemoryLabels`.
- **Calls:** Not inferable from this file.
- **Notes:** Activated only when `MEMDEBUG` is defined; currently `#undef`'d, so debug path is inactive by default.

### FreeMemory
- **Signature:** `void FreeMemory(void *ptr)`
- **Purpose:** Release a previously allocated block back to the custom heap.
- **Inputs:** `ptr` — pointer returned by `GetMemory`/`GetClearedMemory`.
- **Outputs/Return:** void.
- **Side effects:** Modifies internal heap state; decrements tracked total.
- **Calls:** Not inferable from this file.

### MemorySize / TotalAllocatedMemory / PrintMemorySize
- **Notes:** Trivial utility queries — `MemorySize` returns the byte size of a specific allocation, `TotalAllocatedMemory` returns the running total of all live allocations, and `PrintMemorySize` formats a size value for human-readable output. All defined elsewhere.

## Control Flow Notes
This is a header-only declaration file; it participates in compilation-time setup only. The BSPC tool (a BSP compiler/processor) uses these routines throughout its processing pipeline in place of raw `malloc`/`free`, enabling centralized tracking. No frame or tick lifecycle applies — BSPC is a batch offline tool.

## External Dependencies
- No includes within this header.
- **Defined elsewhere:** `GetMemory`, `GetClearedMemory`, `FreeMemory`, `MemorySize`, `PrintMemorySize`, `TotalAllocatedMemory`, and their debug counterparts — all implemented in `code/bspc/l_mem.c`.
