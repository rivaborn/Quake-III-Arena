# common/threads.h — Enhanced Analysis

## Architectural Role
This header provides thread pool infrastructure **exclusively for offline tools** (`q3map`, `bspc`, `q3radiant`), not the runtime engine. It abstracts platform-specific threading (pthreads on Unix, Win32 threads on Windows) behind a simple work-queue model, allowing tools to parallelize CPU-bound tasks like BSP compilation and AAS file generation across all available CPU cores.

## Key Cross-References

### Incoming (who depends on this file)
- **q3map** compiler: uses thread pool for surface preprocessing, lightmap tracing, and visibility computation
- **bspc** AAS compiler: uses thread pool for area cluster computation and reachability analysis
- **q3radiant** level editor: uses thread pool for geometry processing and preview rendering
- **common/threads.c**: implements all declared functions with platform-specific mutexes and thread creation

### Outgoing (what this file depends on)
- **Platform layer**: calls to `pthread_create`/`pthread_mutex_*` (Unix) or `CreateThread`/`CreateMutex` (Win32), hidden behind implementation
- **q_shared.h**: for `qboolean` type definition
- No runtime engine dependencies (pure offline tool infrastructure)

## Design Patterns & Rationale

**Work-queue with two dispatch modes:**
- `RunThreadsOnIndividual`: dynamic per-item work stealing via `GetThreadWork()`. Called by tools needing fine-grained load balancing (e.g., lightmap tracing where per-item cost varies).
- `RunThreadsOn`: static block partitioning per thread. Called by tools with uniform work per partition (e.g., cluster analysis).

**Single global lock:** `ThreadLock`/`ThreadUnlock` protect non-thread-safe engine subsystems (memory allocation, file I/O, loose global state). This is a bottleneck by modern standards but acceptable for offline tools where lock contention is infrequent relative to CPU work.

**Why this design was chosen:** Offline tools of the 2005 era needed simple, portable parallelism without complex synchronization logic. The two dispatch modes reflect real tool diversity; the global lock reflects the reality that shared `malloc`/file operations were not yet thread-safe.

## Data Flow Through This File

1. **Tool startup**: `ThreadSetDefault()` probes CPU count, sets `numthreads` global
2. **Parallel dispatch**: Tool calls `RunThreadsOn[Individual](N, progress_flag, func_ptr)`
3. **Worker loop**: Each thread atomically calls `GetThreadWork()` in a loop, processing `func(item_index)` until -1 returned
4. **Critical section**: If work items access file I/O or allocate memory, they call `ThreadLock()` / `ThreadUnlock()`
5. **Synchronization point**: `RunThreadsOn[Individual]` blocks until all threads complete and join

## Learning Notes

**Era-specific patterns:**
- Pre-C++11 threading: no `<thread>`, no atomics, no condition variables — entirely platform-specific APIs wrapped in C functions
- Work-stealing is manual (`GetThreadWork()` as atomic counter), not automatic (no scheduler)
- Global mutex reflects single-threaded memory allocation era; modern allocators are thread-safe

**Contrast with modern practice:**
- Modern game engines use job systems with work-stealing deques and lock-free queues
- Modern tools would use thread-local arenas or per-thread allocators instead of global lock
- The two dispatch modes foreshadow later task granularity choices (fine vs. coarse parallelism)

**Idiomatic to this codebase:**
- Follows Quake convention of minimal abstractions (interface is only what's strictly necessary)
- Platform layer hidden entirely (good separation); users don't see pthread/Win32 details
- `qboolean` and `int` return types are consistent with Q3 conventions

## Potential Issues

**None clearly inferable from header alone**, but implied concerns:
- **Global lock bottleneck**: Critical sections that hold the lock while doing substantive work (file I/O, allocations) will serialize despite parallelism elsewhere
- **No condition variables**: `RunThreadsOn[Individual]` likely uses busy-wait or sleep loops in implementation; modern code would use CV for efficient thread joining
- **Static `numthreads`**: Assumes single parallel region per process; cannot nest or compose independent parallel tasks

---

**Summary:** A minimalist, platform-agnostic thread pool interface well-suited to offline tool parallelism circa 2005. Its simplicity is a strength for its constrained use case but reflects pre-modern threading best practices.
