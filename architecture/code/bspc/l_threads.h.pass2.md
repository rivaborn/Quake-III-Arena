# code/bspc/l_threads.h — Enhanced Analysis

## Architectural Role

This header provides the threading abstraction for **BSPC** (BSP-to-AAS offline compiler), an isolated tool that generates bot navigation data (`*.aas` files) from compiled BSP maps. The threading API parallelizes the AAS geometry analysis, reachability computation, and clustering phases — which are computationally expensive and embarrassingly parallel. Unlike the runtime engine (which has optional SMP for rendering), BSPC uses threading to accelerate offline preprocessing, not real-time frame loops. The tool runs standalone via command-line and produces binary artifacts consumed by the runtime botlib system.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/bspc/bspc.c`** — Main BSPC entry point; calls `ThreadSetDefault()` during initialization to configure thread pool
- **`code/bspc/aas_*.c`** family (create, cluster, reach, route, optimize, etc.) — All major AAS compilation phases dispatch parallel work via `RunThreadsOn` / `RunThreadsOnIndividual`, passing per-phase callbacks to `GetThreadWork()`
- **`code/bspc/be_aas_bspc.c`** — Higher-level AAS pipeline coordinator; orchestrates threading around reachability, clustering, and optimization passes

### Outgoing (what this file depends on)

- **Platform layer** (`code/unix/l_threads.c`, `code/win32/` threads, `code/null/l_threads.c`) — Concrete mutex/semaphore/thread creation implementations (pthread, Win32 threading API, or stub single-threaded)
- **`code/bspc/l_memory.c`** (inferred) — Likely allocates thread state via internal heap or allocator
- **Platform mutex/semaphore primitives** — Win32 `CreateMutex` / `CreateSemaphore`; POSIX `pthread_mutex_*` / `sem_*`

## Design Patterns & Rationale

**Work-Queue Pattern (`GetThreadWork`):**
- All threads call a shared `GetThreadWork()` to atomically fetch the next work item index from a global counter (guarded by mutex internally)
- Allows dynamic load-balancing: slower threads grab more items, faster threads complete sooner
- Avoids per-thread static work allocation, which would suffer if item processing times vary

**Dual Dispatch API:**
- `RunThreadsOnIndividual(workcnt, ...)`: Work-queue mode — each thread fetches indices 0..N via `GetThreadWork()` until -1
- `RunThreadsOn(workcnt, ...)`: Thread-indexed mode — thread *i* processes indices based on thread ID (likely `i * (workcnt / numthreads)`)
- Rationale: Some phases (e.g., reach computation per-area) benefit from thread-local state; others (e.g., finalization) are sequential but use N threads for parallel sub-work

**Mutex + Semaphore Layer:**
- Mutex guards critical sections (work counter, routing cache updates)
- Semaphore used for producer–consumer coordination (threads wait until main thread signals work is ready, or vice versa)
- Single global mutex/semaphore pair is simpler than per-subsystem locks and acceptable for offline tool (not low-latency real-time)

**Rationale for offline tool design:**
BSPC runs once per map, so startup/shutdown cost is amortized. No need for lock-free data structures or fine-grained synchronization — simplicity and correctness over micro-optimization.

## Data Flow Through This File

```
Input:  numthreads (CLI arg or CPU count)
        workcnt (item count for current phase)
        func (per-item or per-thread callback)

Flow:   ThreadSetDefault()  → detect CPU count, initialize numthreads
        ThreadSetupLock()   → allocate/init global mutex
        ThreadSetupSemaphore() → allocate/init global semaphore
        
        [For each compilation phase]:
        RunThreadsOn[Individual](workcnt, func)
          → spawn numthreads worker threads
          → each thread loops: idx = GetThreadWork() → func(idx) → repeat until -1
          → [mutex guards work counter; semaphore optional for phase handshake]
          → join all threads (blocks main)
        
        ThreadShutdownSemaphore() → cleanup
        ThreadShutdownLock() → cleanup

Output: Per-phase computation complete (e.g., reachability links computed, areas merged, routing cached)
```

## Learning Notes

**Idiomatic to Q3A era (2000s):**
- Simple thread pool with global work counter — common before job schedulers and work-stealing queues
- Portable abstraction (Win32/POSIX behind a header) — no C++11 `<thread>` or modern async APIs
- Offline tool model: compile-once-per-map vs. real-time engine loops

**Modern engines would do differently:**
- Task/job system with hierarchical dependencies (e.g., Unreal Engine 5's Nanite, or custom schedulers in AAA studios)
- Lock-free work-stealing queues per thread (better cache locality, less contention)
- Async/await patterns for phase sequencing rather than explicit join
- Optional GPU acceleration for geometry analysis

**Connection to engine architecture:**
- BSPC is *completely decoupled* from the runtime engine (separate executable, offline)
- Runtime botlib (`code/botlib/`) loads the pre-computed `.aas` file; never re-runs compilation
- Threading in BSPC is internal tool infrastructure; not visible to gameplay or networking code

## Potential Issues

- **Global `numthreads` not re-entrant**: If BSPC later supported batch compilation or live map loading, modifying `numthreads` mid-run would race without additional locking
- **Single global mutex bottleneck**: All critical sections contend for one lock; work-queue counter and any shared data structures (e.g., reachability heap) serialize through it. Fine for offline tool, but would cause SMP scalability cliff with many cores
- **Platform-specific implementation differences**: POSIX semaphore semantics (named vs. unnamed) and Win32 event/mutex type choices could cause subtle synchronization bugs if implementations differ
- **No work-stealing or load-balancing at phase boundaries**: If one phase's work is imbalanced (e.g., 10 heavy items, 990 trivial items), thread pool cannot migrate threads to next phase; main thread stalls after phase completes
