# code/bspc/l_mem.c — Enhanced Analysis

## Architectural Role

This file is the foundational memory management layer for the BSPC offline tool, enabling all BSP compilation and AAS (Area Awareness System) preprocessing to proceed without manual allocation bookkeeping. Unlike the runtime engine (which uses multi-stage hunk/zone allocation), BSPC is a single-pass tool that allocates throughout the compile phase and cleans up via `Hunk_ClearHigh()` / `DumpMemory()` at shutdown. The file bridges BSPC's own allocation needs (map parsing, BSP generation, AAS compilation) with the botlib AAS subsystem, which expects `GetMemory()`, `Hunk_Alloc()`, and `Z_Malloc()` to be available.

## Key Cross-References

### Incoming (who depends on this file)
- **BSPC tool core**: All modules under `code/bspc/` call `GetMemory` / `GetClearedMemory` (e.g., `bspc.c`, `map_q3.c`, `aas_create.c`, `aas_file.c`)
- **botlib AAS pipeline** (shared with runtime): `code/botlib/be_aas_main.c`, `be_aas_file.c`, `be_aas_reach.c` call `GetMemory` when BSPC is compiling AAS; botlib has its own identical `l_memory.c` for runtime use
- **Logging infrastructure**: `l_log.h` functions (`Log_Print`, `Log_Write`) consume output from `PrintMemorySize()`, `PrintMemoryLabels()`, `PrintUsedMemorySize()`
- **Error handling**: `Error()` (from `l_cmd.c`) is called on allocation failure or invalid block pointers

### Outgoing (what this file depends on)
- **C standard library**: `malloc()`, `free()`, `memset()`
- **Windows platform API** (conditional): `_msize()` for heap query (used only in release mode on Win32)
- **Logging**: `Log_Print()`, `Log_Write()` from `l_log.c`
- **Error**: `Error()` from `l_cmd.c` (fatal error handler)

## Design Patterns & Rationale

**Dual-mode allocation**: The `#ifndef MEMDEBUG` / `#else` split creates two completely independent memory managers. 
- **Release mode**: Minimal overhead—just `malloc` wrapping with a global size counter (`allocedmemory`). Calls `MemorySize()`, which uses platform APIs (`_msize` on Win32, always 0 elsewhere) to track actual heap block sizes. Suitable for tool deployment where speed matters and memory bloat is acceptable.
- **Debug mode**: Full instrumentation via prepended `memoryblock_t` headers (magic ID `0x12345678`, user pointer, size, optional file/line/label metadata). Maintains a doubly-linked list and counters (`totalmemorysize`, `numblocks`) for leak detection and validation. Slower but catches corruption early.

**Block metadata prepending**: Rather than out-of-band tracking (hash tables, external registries), metadata is stored directly before the user pointer. This ensures pointer validity checks (`BlockFromPointer`) and prevents orphaned blocks. The magic ID sentinel (`MEM_ID = 0x12345678l`) detects double-free and memory-corruption bugs.

**Hunk compatibility shims**: `Hunk_Alloc()` and `Z_Malloc()`/`Z_Free()` are thin wrappers over the core allocator, allowing BSPC to host botlib code without modification. The hunk maintains a fixed 16 MB budget decremented on each allocation; `Hunk_ClearHigh()` resets it. This models the Quake 3 engine's hunk semantics even though BSPC doesn't actually need a segmented allocator.

## Data Flow Through This File

1. **Initialization**: Global pointers (`memory`, `memhunk_high`, etc.) and counters (`allocedmemory`, `totalmemorysize`) start at NULL/0.

2. **Per-allocation lifecycle**:
   - BSPC tool or botlib calls `GetMemory(N)` or `GetClearedMemory(N)`
   - In debug mode: allocate `N + sizeof(memoryblock_t)` bytes, prepend header with magic ID and metadata, link into `memory` list, increment `totalmemorysize` and `numblocks`
   - In release mode: allocate `N` bytes via `malloc`, query actual size with `MemorySize()`, increment `allocedmemory`
   - Return pointer to usable region

3. **During compilation**: Total tracked memory and block count grow monotonically as BSP/AAS data structures are built.

4. **At shutdown**:
   - `DumpMemory()` iterates the `memory` list (debug) or `Hunk_ClearHigh()` frees the hunk list
   - `PrintMemoryLabels()` or `PrintUsedMemorySize()` dumps final statistics to log
   - All blocks are freed; accounting globals reset

## Learning Notes

**Single-threaded tool design**: BSPC processes one map per invocation, allocates heavily during compilation, and shuts down cleanly. No concurrent access, no reuse across frames, no fragmentation recovery needed. This allows the simple global-list-based approach.

**Sentinel-based corruption detection**: The magic ID (`0x12345678`) is a classic technique for detecting:
- Use-after-free (block's magic gets overwritten)
- Out-of-bounds writes adjacent to the block
- Heap metadata corruption
Combined with `BlockFromPointer`'s pointer-consistency check, it catches many bugs that silent malloc corruption would hide.

**Compilation-time botlib reuse**: BSPC and the runtime engine share botlib source code (`code/botlib/be_aas_*.c`). BSPC compiles botlib offline to generate `.aas` files; the runtime loads them. Both need identical AAS semantics and allocation patterns, so both have `l_memory.c` providing the same interface. This is a form of build-time abstraction.

**Platform API idiosyncrasy**: On non-Windows, `MemorySize()` returns 0, causing release-mode accounting to be meaningless (all allocations are counted as size 0). This suggests the code was debugged primarily on Windows; non-Windows release builds would silently lose accounting fidelity.

## Potential Issues

1. **Broken accounting on non-Windows release mode**: The release-mode `MemorySize()` uses `_msize()` on Windows but returns 0 elsewhere. This causes `allocedmemory` to accumulate 0-byte increments, defeating the purpose of `TotalAllocatedMemory()`. On Linux/Unix, accurate heap accounting is unavailable via standard C, but `malloc_usable_size()` (glibc extension) or `dlmalloc_usable_size()` could help.

2. **Unbounded hunk budget without overflow checking**: `Hunk_Alloc()` decrements `memhunk_high_size` without checking for underflow. If the tool requests more than 16 MB total hunk allocation, `memhunk_high_size` becomes negative (as a signed int), but no error is raised. Subsequent queries (`Hunk_Alloc(0)` to get remaining budget) return garbage. Should add a check and call `Error()` on overflow.

3. **Unused memhunk_low**: The `memhunk_low` and `memhunk_low_size` globals are initialized but never used. Vestigial code from Quake 3 engine porting; can be deleted.

4. **No thread-safety**: All globals are unprotected. Not a problem for a single-threaded tool, but if BSPC ever gains threading (e.g., parallel BSP/AAS compilation), this would need mutexes.

5. **BlockFromPointer dereferencing risk**: If a buggy caller passes a pointer that is not actually from this allocator (e.g., a stack address, a different malloc, or a NULL passed to `FreeMemory`), `BlockFromPointer` will dereference random memory at offset `-(sizeof(memoryblock_t))`, reading whatever is there as a `memoryblock_t` struct. The ID check prevents most false positives, but if the memory happens to contain `0x12345678`, corruption ensues. Could add a sanity check (e.g., ensure `block->ptr == ptr`, which it does, but could validate bounds).

---

**Co-Authored-By**: Claude Haiku 4.5 <noreply@anthropic.com>
