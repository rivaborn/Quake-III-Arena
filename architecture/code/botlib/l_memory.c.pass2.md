# code/botlib/l_memory.c — Enhanced Analysis

## Architectural Role
This file implements botlib's **memory abstraction boundary**, intercepting all allocations from botlib's subsystem modules (AAS navigation, AI pathfinding, reach computation, utility libraries) and delegating to the engine's `botlib_import_t` function table. It enforces a strict dependency inversion: botlib never allocates directly from the OS; all physical allocations come from the engine's zone/hunk, which are themselves managed by `qcommon`. This design enables botlib to be a self-contained, memory-sandboxed module that can be loaded/unloaded independently.

## Key Cross-References

### Incoming (who depends on this file)
- **All botlib subsystem modules** (`be_aas_*.c`, `be_ai_*.c`, `l_*.c`) call `GetMemory`, `GetClearedMemory`, `GetHunkMemory`, `GetClearedHunkMemory`, and `FreeMemory` whenever they allocate state (AAS world, reach caches, routing caches, entity links, string tables, etc.). The first-pass doc notes 60+ such callsites across the botlib codebase.
- **Server frame loop** (`code/server/sv_bot.c`) indirectly depends on this: it calls `trap_BotLib*` syscalls, which eventually cause AAS initialization/loading that triggers large `GetHunkMemory` allocations in `AAS_LoadFiles` → `AAS_LoadAASLump`.
- **botlib initialization** (`be_interface.c`) exports `botlib_export_t` to the engine; the engine initializes botlib by calling `BotLibSetup` which calls `AAS_LoadMap`, which calls `GetHunkMemory`.

### Outgoing (what this file depends on)
- **`botlib_import_t botimport`** (global from `be_interface.c`): Provides the physical allocators `GetMemory`, `FreeMemory`, `HunkAlloc`, `AvailableMemory`, and `Print`. These are function pointers filled by the server at init time.
- **`qcommon` shared code**: `Com_Memset` (from `q_shared.c`) for zeroing buffers in `GetClearedMemory` and `GetClearedHunkMemory`.
- **`l_log.h`**: `Log_Write` (in MEMORYMANAGER mode) for debug logging of all live allocations.
- Implicitly depends on `qcommon`'s zone/hunk memory model: assumes `botimport.GetMemory` points to `Z_Malloc` and `botimport.HunkAlloc` points to `Hunk_Alloc`, both of which enforce the idtech3 two-pool architecture.

## Design Patterns & Rationale

**Dual-Mode Configuration** (`MEMORYMANAGER` vs lightweight): The file ships with two orthogonal implementations compiled conditionally. The full manager mode (when `MEMORYMANAGER` is defined) tracks all live blocks in a doubly-linked list, maintains running counters (total bytes, block count), and can dump diagnostics at shutdown. Lightweight mode only stores a magic word, reducing header overhead but sacrificing introspection. This reflects era-typical embedded-systems thinking: ship two binaries with different memory footprints, letting the team choose at compile-time based on performance/debug needs.

**Magic ID Validation** (`MEM_ID`/`HUNK_ID`): Rather than relying on language-level bounds checking (which C doesn't provide), allocations are tagged with a magic number `0x12345678`. On free, the code recovers the header pointer and checks the magic. This pattern is defensive against buffer overruns that corrupt the header, catching such errors before the free-list corruption propagates. Hunk allocations use a different magic (`HUNK_ID`) to distinguish them; hunk blocks are intentionally not freed, matching the arena allocator semantics.

**Two-Tier Memory Hierarchy**: The existence of both `GetMemory` (heap) and `GetHunkMemory` (arena) mirrors idtech3's zone/hunk dual-pool model. Hunk allocations are treated as permanent (freed only at shutdown via `DumpMemory`), while zone allocations can be freed individually. AAS geometry and routing data live in the hunk (permanent for the map lifetime); transient per-frame data (entity links, routing caches) live in the zone.

**Conditional Debug Instrumentation**: The `MEMDEBUG` mode adds label, file, and line context to all allocations, printed by `PrintMemoryLabels`. This is pre-JSON-logging debugging, using a dedicated log file (`Log_Write`) to track all memory operations. Modern engines would use tagged allocators or memory pools; Q3A uses a flat trace log.

## Data Flow Through This File

1. **Allocation phase** (map load / runtime):
   - `AAS_LoadAASFile` → `GetHunkMemory(size)` → prepend `memoryblock_t` header (manager mode) or `MEM_ID` word (lightweight) → return user pointer
   - Per-bot AI code → `GetClearedMemory(size)` → same flow + `Com_Memset` zero
   
2. **Tracking phase** (if `MEMORYMANAGER`):
   - New block linked into global `memory` linked list
   - Global counters (`allocatedmemory`, `totalmemorysize`, `numblocks`) incremented
   
3. **Deallocation phase**:
   - User passes pointer to `FreeMemory(ptr)`
   - Code recovers header, validates magic ID
   - If heap allocation (`MEM_ID`), calls `botimport.FreeMemory(block)`
   - If hunk allocation (`HUNK_ID`), intentionally skips free (no-op)
   - Counters decremented (manager mode)

4. **Diagnostic phase** (shutdown):
   - `DumpMemory` iterates `memory` list, frees all blocks
   - `PrintMemoryLabels` logs all live blocks to trace file (manager mode)

## Learning Notes

**Defensive Memory Management (2000s era)**: This code reflects pre-modern memory safety practices. No RAII, no garbage collection, no address sanitizers. Instead: magic-word validation, explicit linked-list tracking, and log-based auditing. Modern engines use memory pools, arenas, or generational GC. Q3A's approach is labor-intensive but transparent: you can inspect every byte by reading the trace log.

**Idiomatic idtech3 Pattern**: The zone/hunk dual-pool, magic-word validation, and per-map loading/unloading cycle are idiomatic to id's engine lineage (Quake → Q2 → Q3A → id Tech 4). Understanding this file teaches you how engines of that era managed memory: via explicit two-stage allocation with arena properties and defensive validation.

**Library Sandbox**: By routing all allocations through `botlib_import_t`, the engine can precisely measure botlib's memory footprint (via `AvailableMemory` calls) and even substitute mock allocators for testing. This is an inversion-of-control pattern that predates dependency injection frameworks.

**Asymmetric Free semantics**: The fact that hunk allocations are *never freed* (except at `DumpMemory` shutdown) reflects a deliberate design choice: hunk memory is trusted to outlive any individual botlib subsystem. This is safe because the map doesn't change during gameplay; all hunk data is valid for the entire map lifetime. Heap allocations, by contrast, are freed immediately when no longer needed. This asymmetry is not a bug; it's a performance optimization for a memory allocator that expects large contiguous allocations and infrequent free operations.

## Potential Issues

**No bounds checking on reallocation**: The file provides no `Realloc` function; code that needs to grow a buffer must allocate a new block, copy, and free the old one. This is inefficient but avoids complex pointer-tracking in manager mode.

**Manager mode overhead is significant**: Each allocation requires a `memoryblock_t` structure (56 bytes in manager mode with debug info). For botlib's thousands of small allocations (reach links, entity cache entries), this overhead is substantial. The lightweight mode exists precisely to reduce this.

**No thread safety**: The global `memory` list and counters (`allocatedmemory`, etc.) are not protected by locks. This is acceptable because botlib initialization happens on the server thread before per-frame updates, and per-frame AAS calls don't allocate. However, concurrent initialization or multi-threaded bots would require synchronization.

**Hunk allocation is one-way**: Once allocated from the hunk, memory never shrinks. If an AAS file is replaced mid-session (unlikely but possible), the old hunk data persists until `DumpMemory`. This is a memory leak by design—the hunk was never meant to support hot-swapping.
