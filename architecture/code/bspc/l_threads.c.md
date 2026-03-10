# code/bspc/l_threads.c

## File Purpose
Provides a cross-platform threading abstraction layer for the BSPC (BSP compiler) tool. It implements mutexes, semaphores, work dispatch, and thread lifecycle management with four platform-specific backends: Win32, OSF1 (Digital Unix), Linux (pthreads), and IRIX (sproc), plus a no-op single-threaded fallback.

## Core Responsibilities
- Dispatch a fixed work queue across N worker threads with progress reporting
- Provide mutex (ThreadLock/ThreadUnlock) and semaphore primitives per platform
- Manage a linked list of dynamically spawned threads (AddThread/RemoveThread)
- Auto-detect CPU count for default thread count (Win32, IRIX)
- Gate all multi-threaded paths behind the `threaded` flag to catch misuse
- Provide `RunThreadsOnIndividual` as a higher-level wrapper that assigns one work item per thread invocation

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `thread_t` | struct (per-platform) | Linked-list node tracking a spawned thread: handle/pthread, threadid, sequential id, next pointer |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `dispatch` | `int` | global | Next work-item index to hand out |
| `workcount` | `int` | global | Total number of work items in current batch |
| `oldf` | `int` | global | Last printed progress decile (0–9), avoids duplicate prints |
| `pacifier` | `qboolean` | global | Whether to print progress dots |
| `threaded` | `qboolean` | global | Guards lock/unlock calls; set true only during multi-threaded runs |
| `workfunction` | `void(*)(int)` | global | Per-item callback used by `ThreadWorkerFunction` |
| `numthreads` | `int` | global (per-platform block) | Configured thread count; -1 = auto-detect |
| `firstthread` / `lastthread` | `thread_t *` | global | Head/tail of active thread linked list |
| `currentnumthreads` | `int` | global | Count of currently live threads |
| `currentthreadid` | `int` | global | Monotonically incrementing id for next thread |
| `enter` | `static int` | static | Reentrancy guard for ThreadLock (detects recursive lock) |
| `crit` (Win32) | `CRITICAL_SECTION` | global | Win32 mutex object |
| `semaphore` (Win32/Linux) | `HANDLE` / `sem_t` | global | Platform semaphore object |
| `my_mutex` (OSF1/Linux) | `pthread_mutex_t` | global | POSIX mutex |
| `lck` (IRIX) | `abilock_t` | global | IRIX spin-lock |

## Key Functions / Methods

### GetThreadWork
- **Signature:** `int GetThreadWork(void)`
- **Purpose:** Thread-safe fetch of the next work index from the shared queue, with 10-step progress printing.
- **Inputs:** None (reads `dispatch`, `workcount`, `oldf`, `pacifier`)
- **Outputs/Return:** Next work index, or `-1` when all work is consumed
- **Side effects:** Increments `dispatch`; prints progress to stdout if `pacifier` is set; acquires/releases thread lock
- **Calls:** `ThreadLock`, `ThreadUnlock`, `printf`
- **Notes:** The `f = 10*dispatch/workcount` progress decile is computed before incrementing, so the last item may not print "9".

### ThreadWorkerFunction
- **Signature:** `void ThreadWorkerFunction(int threadnum)`
- **Purpose:** Entry point for worker threads launched by `RunThreadsOnIndividual`; loops calling `GetThreadWork` then dispatching to `workfunction`.
- **Inputs:** `threadnum` — thread index (unused beyond identification)
- **Outputs/Return:** void
- **Side effects:** Repeatedly invokes `workfunction` with work indices until queue is empty
- **Calls:** `GetThreadWork`, `workfunction`

### RunThreadsOnIndividual
- **Signature:** `void RunThreadsOnIndividual(int workcnt, qboolean showpacifier, void(*func)(int))`
- **Purpose:** High-level entry that wraps `RunThreadsOn`, using `ThreadWorkerFunction` to map one work item per call to `func`.
- **Inputs:** `workcnt` — item count; `showpacifier` — progress flag; `func` — per-item callback
- **Side effects:** Sets `workfunction`; calls `ThreadSetDefault` if needed
- **Calls:** `ThreadSetDefault`, `RunThreadsOn`

### RunThreadsOn (per-platform)
- **Signature:** `void RunThreadsOn(int workcnt, qboolean showpacifier, void(*func)(int))`
- **Purpose:** Spawns N threads all running `func`, waits for completion, measures wall time.
- **Inputs:** `workcnt` — total work items; `showpacifier`; `func` — thread entry (receives thread index)
- **Side effects:** Sets/clears `threaded`; initializes/destroys platform mutex; prints elapsed time
- **Calls (Win32):** `InitializeCriticalSection`, `CreateThread`, `WaitForSingleObject`, `DeleteCriticalSection`, `I_FloatTime`, `Log_Print`
- **Notes:** Falls through to single-call `func(0)` when `numthreads == 1` (Win32 path). Single-thread fallback `#ifndef USED` version calls `func(0)` directly.

### AddThread / RemoveThread
- **Signature:** `void AddThread(void(*func)(int))` / `void RemoveThread(int threadid)`
- **Purpose:** Dynamically spawn or join a thread into the persistent linked list; used for producer/consumer patterns distinct from `RunThreadsOn`.
- **Side effects:** Alloc/free `thread_t` via `GetMemory`/`FreeMemory`; modifies `firstthread`/`lastthread`; modifies `currentnumthreads`
- **Notes:** `threadid == -1` in `RemoveThread` signals the single-thread path (no-op).

### WaitForAllThreadsFinished
- **Signature:** `void WaitForAllThreadsFinished(void)`
- **Purpose:** Block until the `firstthread` linked list is empty, joining each thread handle in turn.
- **Notes:** Releases lock before blocking join to avoid deadlock; re-acquires for list traversal.

## Control Flow Notes
This file is a pure utility used during the BSP compilation offline tool pipeline. It is not part of the game runtime. `RunThreadsOnIndividual` / `RunThreadsOn` are called from compute-heavy BSP stages (visibility, lighting, AAS generation) to parallelize work. There is no frame loop involvement.

## External Dependencies
- `l_cmd.h` — `Error`, `qprintf`, `I_FloatTime`, `qboolean`
- `l_threads.h` — declares all exported symbols
- `l_log.h` — `Log_Print`
- `l_mem.h` — `GetMemory`, `FreeMemory`
- **Win32:** `<windows.h>` — `CRITICAL_SECTION`, `CreateThread`, `WaitForSingleObject`, `CreateSemaphore`, `ReleaseSemaphore`, `GetSystemInfo`
- **OSF1/Linux:** `<pthread.h>` — `pthread_create`, `pthread_join`, `pthread_mutex_*`
- **Linux:** `<semaphore.h>` — `sem_init`, `sem_wait`, `sem_post`, `sem_destroy`
- **IRIX:** `<task.h>`, `<abi_mutex.h>`, `<sys/prctl.h>` — `sprocsp`, `spin_lock`, `release_lock`, `init_lock`
