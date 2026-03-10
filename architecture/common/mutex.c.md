# common/mutex.c

## File Purpose
Provides a thin, platform-abstracted mutex API used by the Quake III build tools (q3map, bspc, etc.) to protect shared state during multi-threaded work distribution. Exactly one platform implementation is compiled via preprocessor guards, with a no-op fallback for single-threaded or unsupported builds.

## Core Responsibilities
- Allocate and initialize platform-native mutex objects (`MutexAlloc`)
- Acquire a mutex lock (`MutexLock`)
- Release a mutex lock (`MutexUnlock`)
- Short-circuit all locking when `numthreads == 1` (returns `NULL`, lock/unlock ignore `NULL`)
- Provide a do-nothing fallback when no recognized platform is detected

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `mutex_t` | typedef (`void *`) | Opaque handle to a platform-native mutex; defined in `mutex.h` |

Platform-internal types (not exported):

| Platform | Underlying Type |
|----------|----------------|
| WIN32 | `CRITICAL_SECTION` |
| OSF1 (`__osf__`) | `pthread_mutex_t` |
| IRIX (`_MIPS_ISA`) | `abilock_t` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `numthreads` | `int` | global (defined in `threads.c`) | Read by `MutexAlloc` to skip allocation when single-threaded |

## Key Functions / Methods

### MutexAlloc
- **Signature:** `mutex_t *MutexAlloc(void)`
- **Purpose:** Allocates and initializes a platform-native mutex object.
- **Inputs:** None.
- **Outputs/Return:** Opaque `mutex_t *` pointer to the allocated mutex, or `NULL` if `numthreads == 1`.
- **Side effects:** `malloc` of a platform mutex struct; initializes the OS primitive (`InitializeCriticalSection` / `pthread_mutex_init` / `init_lock`). OSF1 path calls `Error()` on init failure, terminating the process.
- **Calls:** `malloc`, `InitializeCriticalSection` (WIN32); `malloc`, `pthread_mutexattr_create`, `pthread_mutexattr_setkind_np`, `pthread_mutex_init`, `Error` (OSF1); `malloc`, `init_lock` (IRIX).
- **Notes:** Callers must check for `NULL` return before use (single-thread case). No corresponding `MutexFree` exists in this file — allocated mutexes are never explicitly freed (tool lifetime = process lifetime).

### MutexLock
- **Signature:** `void MutexLock(mutex_t *m)`
- **Purpose:** Acquires exclusive ownership of the mutex.
- **Inputs:** `m` — opaque mutex handle; `NULL` is silently ignored (single-thread path).
- **Outputs/Return:** `void`.
- **Side effects:** Blocks the calling thread until the lock is acquired.
- **Calls:** `EnterCriticalSection` (WIN32); `pthread_mutex_lock` (OSF1); `spin_lock` (IRIX).
- **Notes:** IRIX implementation is a spin-lock (`spin_lock`), not a sleeping lock — busy-waits under contention.

### MutexUnlock
- **Signature:** `void MutexUnlock(mutex_t *m)`
- **Purpose:** Releases exclusive ownership of the mutex.
- **Inputs:** `m` — opaque mutex handle; `NULL` is silently ignored.
- **Outputs/Return:** `void`.
- **Side effects:** Wakes any thread blocked in `MutexLock`.
- **Calls:** `LeaveCriticalSection` (WIN32); `pthread_mutex_unlock` (OSF1); `release_lock` (IRIX).
- **Notes:** Calling `MutexUnlock` without a prior `MutexLock` is undefined behavior at the OS level; this file adds no guard against it.

## Control Flow Notes
This file is a build-tool utility, not part of the game runtime. It is used during offline map/BSP compilation (q3map, bspc). `MutexAlloc` is called at tool startup when multi-threaded work pools are initialized. `MutexLock`/`MutexUnlock` are called around shared data accesses during parallel work dispatch (see `threads.c`). There is no shutdown path — OS cleans up on process exit.

## External Dependencies
- `cmdlib.h` — provides `Error()` (fatal error termination), used on OSF1 mutex init failure
- `threads.h` — provides `numthreads` global
- `mutex.h` — declares `mutex_t` and the three public function prototypes
- `<windows.h>` (WIN32) — `CRITICAL_SECTION`, `EnterCriticalSection`, `LeaveCriticalSection`, `InitializeCriticalSection`
- `<pthread.h>` (OSF1) — `pthread_mutex_t`, `pthread_mutex_lock/unlock/init`, `pthread_mutexattr_*`
- `<task.h>`, `<abi_mutex.h>`, `<sys/types.h>`, `<sys/prctl.h>` (IRIX) — `abilock_t`, `spin_lock`, `release_lock`, `init_lock`
