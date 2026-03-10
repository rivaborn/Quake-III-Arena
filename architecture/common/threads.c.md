# common/threads.c

## File Purpose
Provides a platform-abstracted threading layer for the Quake III build tools (q3map, bspc, etc.). It implements a work-queue dispatcher pattern where multiple threads pull integer work items from a shared counter, with compile-time backends for Win32, OSF1, IRIX, and a single-threaded fallback.

## Core Responsibilities
- Maintain a global work-item dispatch counter shared across all threads
- Provide `ThreadLock`/`ThreadUnlock` around the dispatch counter (platform-specific mutex/critical section)
- Report progress as a 0–9 percentage pacifier during long operations
- Spawn and join N worker threads via `RunThreadsOn`, dispatching a callback per item
- Provide `RunThreadsOnIndividual` as a higher-level wrapper that sets `workfunction` and delegates
- Auto-detect thread count from hardware (Win32/IRIX) or use a fixed default (OSF1)

## Key Types / Data Structures

None (no structs or typedefs defined in this file).

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `dispatch` | `int` | global | Next work index to hand out |
| `workcount` | `int` | global | Total number of work items |
| `oldf` | `int` | global | Last printed progress fraction (0–9) |
| `pacifier` | `qboolean` | global | Whether to print progress |
| `threaded` | `qboolean` | global | Whether multithreading is active; gates lock calls |
| `numthreads` | `int` | global (per-platform) | Number of worker threads to spawn |
| `workfunction` | `void (*)(int)` | global | Per-item callback set by `RunThreadsOnIndividual` |
| `crit` (Win32) | `CRITICAL_SECTION` | static | Win32 mutex guarding `dispatch` |
| `enter` (Win32) | `int` | static | Re-entrancy guard for Win32 critical section |
| `my_mutex` (OSF1) | `pthread_mutex_t *` | global | OSF1 pthread mutex (lazily allocated) |
| `lck` (IRIX) | `abilock_t` | global | IRIX spin-lock |

## Key Functions / Methods

### GetThreadWork
- **Signature:** `int GetThreadWork(void)`
- **Purpose:** Atomically claim the next work index; print pacifier progress if enabled.
- **Inputs:** None (reads global `dispatch`, `workcount`, `oldf`, `pacifier`)
- **Outputs/Return:** Next work index `[0, workcount)`, or `-1` when all work is consumed
- **Side effects:** Increments `dispatch`; may call `_printf` for progress; acquires/releases thread lock
- **Calls:** `ThreadLock`, `ThreadUnlock`, `_printf`
- **Notes:** Must be called only while `threaded` is true (or lock calls are no-ops); the 10-bucket progress display rounds down, so "9..." prints at 90–99%.

### ThreadWorkerFunction
- **Signature:** `void ThreadWorkerFunction(int threadnum)`
- **Purpose:** Per-thread entry point; loops calling `GetThreadWork` and dispatching to `workfunction` until no work remains.
- **Inputs:** `threadnum` — thread index (unused beyond loop identity)
- **Outputs/Return:** void
- **Side effects:** Calls `workfunction(work)` repeatedly; terminates when `GetThreadWork` returns -1
- **Calls:** `GetThreadWork`, `workfunction` (indirect)
- **Notes:** The debug `_printf` for thread/work logging is commented out.

### RunThreadsOnIndividual
- **Signature:** `void RunThreadsOnIndividual(int workcnt, qboolean showpacifier, void(*func)(int))`
- **Purpose:** High-level entry point — stores `func` in `workfunction` and delegates to `RunThreadsOn` with `ThreadWorkerFunction` as the thread body.
- **Inputs:** `workcnt` — item count; `showpacifier` — progress flag; `func` — per-item callback
- **Side effects:** Sets global `workfunction`; calls `ThreadSetDefault` if `numthreads == -1`
- **Calls:** `ThreadSetDefault`, `RunThreadsOn`

### RunThreadsOn (Win32)
- **Signature:** `void RunThreadsOn(int workcnt, qboolean showpacifier, void(*func)(int))`
- **Purpose:** Initialize state, spawn `numthreads` Win32 threads running `func`, wait for all to finish, then clean up.
- **Inputs:** `workcnt`, `showpacifier`, `func`
- **Side effects:** Sets/clears globals `dispatch`, `workcount`, `oldf`, `pacifier`, `threaded`; initializes and deletes `crit`; prints elapsed time if pacifier is on
- **Calls:** `I_FloatTime`, `InitializeCriticalSection`, `CreateThread`, `WaitForSingleObject`, `DeleteCriticalSection`, `_printf`
- **Notes:** Single-thread shortcut: when `numthreads == 1`, calls `func(0)` directly without spawning.

### ThreadSetDefault / ThreadLock / ThreadUnlock
- **Notes:** Three platform-specific trios. `ThreadSetDefault` queries hardware concurrency (Win32: `GetSystemInfo`; IRIX: `PR_MAXPPROCS`; OSF1: hardcoded 4; fallback: 1). `ThreadLock`/`ThreadUnlock` are no-ops in the fallback and no-ops when `!threaded` on Win32.

## Control Flow Notes
Used exclusively by offline build tools, not the game runtime. Callers invoke `RunThreadsOnIndividual` (or `RunThreadsOn` directly) during BSP/lighting/vis processing. The file plays no role in the game's init/frame/render/shutdown loop.

## External Dependencies
- `cmdlib.h` — `qboolean`, `Error`, `_printf`, `qprintf`, `I_FloatTime`
- `threads.h` — declares `numthreads`, `GetThreadWork`, `RunThreadsOn`, `RunThreadsOnIndividual`, `ThreadLock`, `ThreadUnlock`, `ThreadSetDefault`
- **Win32:** `<windows.h>` — `CRITICAL_SECTION`, `CreateThread`, `WaitForSingleObject`
- **OSF1:** `<pthread.h>` — `pthread_mutex_t`, `pthread_create`, `pthread_join`
- **IRIX:** `<task.h>`, `<abi_mutex.h>`, `<sys/prctl.h>` — `sprocsp`, `abilock_t`, `spin_lock`/`release_lock`
