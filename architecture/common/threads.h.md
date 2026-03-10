# common/threads.h

## File Purpose
Declares the public interface for the thread management system used by Quake III's offline tools (map compiler, BSP tools). Provides a simple work-queue threading model with a global lock for non-thread-safe operations.

## Core Responsibilities
- Expose thread count configuration via `ThreadSetDefault`
- Distribute discrete work items across threads via a shared work counter
- Provide a mutual exclusion primitive (`ThreadLock`/`ThreadUnlock`) for critical sections
- Abstract platform-specific threading behind a uniform function-pointer dispatch API

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numthreads` | `int` | global (extern) | Number of worker threads to use; set by `ThreadSetDefault` and used by `RunThreadsOn`/`RunThreadsOnIndividual` |

## Key Functions / Methods

### ThreadSetDefault
- **Signature:** `void ThreadSetDefault(void)`
- **Purpose:** Initializes `numthreads` to an appropriate default (typically CPU count or 1 for single-threaded builds)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Writes to global `numthreads`
- **Calls:** Not inferable from this file
- **Notes:** Should be called once at tool startup before any threaded work

### GetThreadWork
- **Signature:** `int GetThreadWork(void)`
- **Purpose:** Atomically retrieves and increments the next work item index from a shared counter; returns -1 when all work is exhausted
- **Inputs:** None
- **Outputs/Return:** `int` — next work index, or -1 if done
- **Side effects:** Modifies internal shared work counter (defined in `threads.c`)
- **Calls:** Not inferable from this file
- **Notes:** Must be called from within a thread dispatch; callers loop until -1 is returned

### RunThreadsOnIndividual
- **Signature:** `void RunThreadsOnIndividual(int workcnt, qboolean showpacifier, void(*func)(int))`
- **Purpose:** Spawns `numthreads` threads; each thread calls `func` once per work item obtained via `GetThreadWork`
- **Inputs:** `workcnt` — total number of work items; `showpacifier` — whether to print progress; `func` — per-item callback
- **Outputs/Return:** None; blocks until all work is complete
- **Side effects:** Thread creation/join, optional stdout progress output
- **Calls:** `GetThreadWork`, `func`
- **Notes:** Differs from `RunThreadsOn` in that work is distributed item-by-item rather than in block ranges

### RunThreadsOn
- **Signature:** `void RunThreadsOn(int workcnt, qboolean showpacifier, void(*func)(int))`
- **Purpose:** Spawns threads and dispatches `func` with a pre-partitioned work range per thread
- **Inputs:** `workcnt` — total work count; `showpacifier` — progress display flag; `func` — worker callback receiving a thread index
- **Outputs/Return:** None; blocks until all threads complete
- **Side effects:** Thread creation/join, optional stdout progress output
- **Calls:** `func`
- **Notes:** Preferred when each thread processes an independent partition; complementary to `RunThreadsOnIndividual`

### ThreadLock / ThreadUnlock
- **Signature:** `void ThreadLock(void)` / `void ThreadUnlock(void)`
- **Purpose:** Acquire and release a single global mutex protecting non-thread-safe code (e.g., memory allocation, file I/O)
- **Inputs:** None
- **Outputs/Return:** None
- **Side effects:** Blocks calling thread until lock is available; modifies internal mutex state
- **Calls:** Not inferable from this file
- **Notes:** Non-reentrant; calling `ThreadLock` twice from the same thread will deadlock

## Control Flow Notes
This header is used exclusively by offline tools (`q3map`, `bspc`, etc.) during build/compile pipelines — not at game runtime. The typical flow is: `ThreadSetDefault` → `RunThreadsOn` or `RunThreadsOnIndividual` → per-item callbacks use `ThreadLock`/`ThreadUnlock` around shared state.

## External Dependencies
- `qboolean` — defined in `q_shared.h` or equivalent shared header; not defined here
- Implementation symbols (`numthreads`, mutex state, work counter) — defined in `common/threads.c`
