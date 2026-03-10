# code/bspc/l_threads.h

## File Purpose
Public header declaring the threading API for the BSPC (BSP compiler) tool. It exposes thread management, work dispatch, mutual exclusion (mutex), and semaphore primitives used during parallel BSP/AAS compilation tasks.

## Core Responsibilities
- Declare the global thread count variable (`numthreads`)
- Expose thread pool initialization and work-queue dispatch functions
- Declare mutex lock/unlock primitives for critical section protection
- Declare semaphore primitives for producer/consumer synchronization
- Declare dynamic thread add/remove and join-all utilities

## Key Types / Data Structures
None. (Header is declaration-only; all types are primitive or defined elsewhere.)

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `numthreads` | `int` | global (`extern`) | Controls how many worker threads are spawned for parallel work dispatch |

## Key Functions / Methods

### ThreadSetDefault
- Signature: `void ThreadSetDefault(void)`
- Purpose: Initializes `numthreads` to a platform-appropriate default (e.g., number of CPUs).
- Inputs: None
- Outputs/Return: None
- Side effects: Writes `numthreads` global
- Calls: Not inferable from this file
- Notes: Should be called before any `RunThreadsOn*` call.

### GetThreadWork
- Signature: `int GetThreadWork(void)`
- Purpose: Atomically fetches the next work item index from a shared work counter; returns -1 when exhausted.
- Inputs: None
- Outputs/Return: `int` — next work index, or -1 if no work remains
- Side effects: Modifies internal work counter (guarded by mutex internally)
- Calls: Not inferable from this file
- Notes: Must be called from within a thread worker function.

### RunThreadsOnIndividual
- Signature: `void RunThreadsOnIndividual(int workcnt, qboolean showpacifier, void(*func)(int))`
- Purpose: Spawns up to `numthreads` threads, each calling `func` with individual work indices obtained via `GetThreadWork`.
- Inputs: `workcnt` — total items; `showpacifier` — display progress; `func` — per-item callback
- Outputs/Return: None (blocks until all work is done)
- Side effects: Thread creation/join, optional console I/O
- Calls: Not inferable from this file

### RunThreadsOn
- Signature: `void RunThreadsOn(int workcnt, qboolean showpacifier, void(*func)(int))`
- Purpose: Similar to `RunThreadsOnIndividual` but passes the thread index rather than a work-queue index to `func`.
- Inputs: Same as `RunThreadsOnIndividual`
- Outputs/Return: None (blocks until all threads finish)
- Side effects: Thread creation/join, optional console I/O
- Notes: Distinction from `RunThreadsOnIndividual` is inferable by name; exact dispatch semantics defined in `l_threads.c`.

### ThreadSetupLock / ThreadShutdownLock
- Signature: `void ThreadSetupLock(void)` / `void ThreadShutdownLock(void)`
- Purpose: Initialize and destroy the global mutex resource.

### ThreadLock / ThreadUnlock
- Signature: `void ThreadLock(void)` / `void ThreadUnlock(void)`
- Purpose: Acquire/release the global mutex for critical sections.
- Side effects: Blocks calling thread if lock is contended.

### ThreadSetupSemaphore / ThreadShutdownSemaphore
- Signature: `void ThreadSetupSemaphore(void)` / `void ThreadShutdownSemaphore(void)`
- Purpose: Initialize and destroy the global semaphore.

### ThreadSemaphoreWait / ThreadSemaphoreIncrease
- Signature: `void ThreadSemaphoreWait(void)` / `void ThreadSemaphoreIncrease(int count)`
- Purpose: Block until semaphore is signaled; increment semaphore by `count`.

### AddThread / RemoveThread / WaitForAllThreadsFinished / GetNumThreads
- `AddThread(void(*func)(int))` — Spawns a new thread executing `func`.
- `RemoveThread(int threadid)` — Terminates or detaches a thread by ID.
- `WaitForAllThreadsFinished(void)` — Blocks until all active threads exit.
- `GetNumThreads(void) → int` — Returns current live thread count.
- Notes: These form a lower-level dynamic thread management layer beneath the higher-level `RunThreadsOn*` API.

## Control Flow Notes
Used exclusively during BSP/AAS compilation (offline tool, not runtime). Typical flow: `ThreadSetDefault` → `ThreadSetupLock` → `ThreadSetupSemaphore` → `RunThreadsOn[Individual]` (which internally calls `GetThreadWork`, `ThreadLock`/`ThreadUnlock`) → `ThreadShutdownLock` / `ThreadShutdownSemaphore`.

## External Dependencies
- `qboolean` — defined in shared Q3 headers (e.g., `q_shared.h`)
- Implementations defined in `code/bspc/l_threads.c` (platform-specific: Win32, POSIX pthreads, or null/single-threaded stub)
