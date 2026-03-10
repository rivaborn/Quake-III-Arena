# code/botlib/l_memory.h — Enhanced Analysis

## Architectural Role

This header is the **memory abstraction layer** for the autonomous botlib subsystem, providing a unified interface that decouples memory allocation strategy from the rest of botlib's 40+ modules. It acts as a facade that masks whether allocations come from the heap or the engine's hunk allocator, allowing the same botlib code to run both as a runtime plugin (integrated with the engine's memory model) and as part of the standalone BSPC offline compiler tool.

## Key Cross-References

### Incoming (who depends on this file)
- **Every botlib .c file** in navigation (be_aas_*.c), AI (be_ai_*.c), and utilities (l_*.c) calls `GetMemory`/`GetClearedMemory`/`GetHunkMemory` for AAS structure allocation, routing cache allocation, and internal bookkeeping
- **be_aas_main.c**, **be_aas_route.c**, **be_aas_reach.c**: Heavy users during initialization to construct AAS cluster/reachability/routing data (allocated once, held for the level lifetime, then freed with `DumpMemory`)
- **l_libvar.c**, **l_script.c**: Utility infrastructure for config and parsing, also allocate temporaries via `GetMemory`
- **BSPC tool** (code/bspc/be_aas_bspc.c): Reuses the same botlib code by conditionally compiling with `#ifdef BSPC` to alias hunk→heap

### Outgoing (what this file depends on)
- **l_memory.c** (implementation, not visible in header): Calls down to whatever engine trap provides hunk memory; on non-BSPC builds the trap is wired via `botlib_import_t botimport.Alloc`
- **qcommon/mem.c / qcommon/vm.c**: The engine's hunk allocator invoked via trap; the implementation of `GetHunkMemory` is `botimport.Alloc(size)` at runtime
- **Standard C library**: In BSPC mode, `GetHunkMemory` → `GetMemory` → direct `malloc`
- Globals read/written: None in this header; actual tracking state lives in l_memory.c

## Design Patterns & Rationale

### 1. **Dual-Mode Compile-Time Polymorphism (BSPC Conditional)**
```c
#ifdef BSPC
#define GetHunkMemory GetMemory
#define GetClearedHunkMemory GetClearedMemory
#else
void *GetHunkMemory(unsigned long size);
void *GetClearedHunkMemory(unsigned long size);
#endif
```
- **Why**: The botlib AAS pipeline (reachability computation, cluster building) is CPU-intensive and used in two contexts:
  - *Runtime*: Botlib is a plugin; the engine provides a hunk allocator for long-lived level data
  - *Offline*: BSPC is a standalone tool with no engine; it must use malloc-based heap
- **Tradeoff**: Single AAS codebase, two allocation strategies, zero vtable overhead. Enabled by preprocessor — no runtime dispatch cost.

### 2. **Debug Instrumentation via Macro Redirection (MEMDEBUG)**
```c
#ifdef MEMDEBUG
#define GetMemory(size) GetMemoryDebug(size, #size, __FILE__, __LINE__);
#else
void *GetMemory(unsigned long size);
#endif
```
- **Why**: Classic debug pattern. Captures the *source expression* (`#size`), filename, and line number *at call site* without changing the caller.
- **Rationale**: Botlib is a complex module. Memory leaks in pathfinding queries or entity tracking are hard to trace. This enables post-mortem leak detection via `PrintMemoryLabels()` and `MemoryByteSize()`.
- **Tradeoff**: Zero cost in release builds (empty define); full overhead in debug builds (extra parameters, shadow vtable).

### 3. **Semantic Separation: Heap vs. Hunk**
The presence of two allocation families hints at botlib's memory lifetime model:
- **Heap** (`GetMemory`, `GetClearedMemory`): Per-query or per-frame allocations (e.g., temporary reachability checks, local entity cache updates). Short-lived, freed individually.
- **Hunk** (`GetHunkMemory`, `GetClearedHunkMemory`): Level-scoped data (the entire AAS structure, routing caches, cluster definitions). Freed wholesale with `DumpMemory()` at level shutdown.

This mirrors the engine's own `Hunk_Alloc` / `malloc` split and allows the engine to tune memory layout for cache locality.

## Data Flow Through This File

1. **Initialization phase** (server startup):
   - Server calls `trap_BotLibSetup` → `BotImportInit` wires `botlib_import_t.Alloc` to engine's hunk allocator
   - `AAS_LoadMap` → `AAS_LoadFiles` → `GetHunkMemory(size)` → `botimport.Alloc(size)`
   - Entire AAS structure (vertices, edges, faces, areas, reachability graph) allocated and held

2. **Per-frame runtime**:
   - Bot AI queries call `AAS_BestReachableArea`, `AAS_AreaRouteToGoal`, etc.
   - These may temporarily allocate via `GetMemory` for intermediate results (e.g., heap-allocated route cache entries)
   - Freed back to heap with `FreeMemory` before frame end

3. **Level shutdown**:
   - `AAS_Shutdown` → `DumpMemory()` releases all hunk blocks in one pass
   - The botlib singleton `aasworld` is zeroed; next level reuses the hunk range

## Learning Notes

### Idiomatic to This Era (Early 2000s Game Engines)
- **No vtables / runtime polymorphism**: Pure preprocessor-based allocation swapping. Zero overhead, maximal static analysis.
- **Dual allocators**: Separation of long-lived (hunk/arena) vs. short-lived (heap) data was standard in the Quake lineage. Modern engines use slab allocators or generational GC instead.
- **Manual memory accounting**: `MemoryByteSize`, `AvailableMemory`, `PrintUsedMemorySize` expose raw statistics. No automatic profiling or adaptive GC.
- **Leak tracking via labels**: `GetMemoryDebug(..., "label", ...)` is a mid-level approach; modern tools use Valgrind or AddressSanitizer, but this was pragmatic for 2000s console/PC development.

### Modern Engines
- Most now use unified allocators (buddy system, buddy trees, or jemalloc) with tagging/profiling built in.
- Dual heap/hunk split is less common; generational arenas replaced it in engines like Unreal Engine 3+.
- However, Q3's pattern of "hunk as scratch/frame buffer" persists in some form in many engines.

### Connection to Broader botlib Architecture
- `l_memory.h` is the **foundation** of botlib's utility layer (l_*.c files).
- It enables `l_libvar.c` (config system), `l_script.c` (text parser), and all the `be_aas_*.c` modules to allocate without knowing *how* or *where*.
- This abstraction is key to botlib's portability: it ships as a self-contained `.a` library that the engine plugs in via `botlib_import_t`.

## Potential Issues

1. **No error handling for allocation failure**: Neither `GetMemory` nor `GetHunkMemory` declares a failure mode. Callers assume `malloc`/hunk never fail (typical for 2000s; modern code should check for NULL). If an allocation fails and returns NULL, botlib code likely seg-faults rather than gracefully degrading.

2. **MEMDEBUG build only tracks allocations, not actual leaks**: `PrintMemoryLabels()` lists live blocks, but there is no automatic leak detection at shutdown. A developer must manually audit the output — error-prone.

3. **No thread safety**: No mutexes or atomics. If botlib is multi-threaded at the VM level (unlikely in Q3 given single-threaded `gvm`), concurrent allocation could corrupt the internal tracking state.

4. **Hunk alias hidden in conditional**: The mapping `GetHunkMemory → GetMemory` under `#ifdef BSPC` is a **silent semantic change**. A developer might not realize that BSPC mode has fundamentally different memory lifetime semantics (no level unload; hunk = heap = malloc).
