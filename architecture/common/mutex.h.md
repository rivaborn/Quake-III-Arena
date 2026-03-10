# common/mutex.h

## File Purpose
Declares a minimal, platform-agnostic mutex abstraction used by the Quake III toolchain (map compiler, BSPC, etc.) to synchronize multi-threaded operations. It provides opaque handle allocation and lock/unlock primitives over whatever threading backend the platform supplies.

## Core Responsibilities
- Define the opaque `mutex_t` handle type
- Declare allocation of a new mutex
- Declare lock and unlock operations on a mutex handle

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `mutex_t` | typedef (opaque pointer — `void *`) | Platform-agnostic handle to an OS mutex object |

## Global / File-Static State
None.

## Key Functions / Methods

### MutexAlloc
- **Signature:** `mutex_t *MutexAlloc(void)`
- **Purpose:** Allocates and initializes a new mutex object, returning an opaque handle to it.
- **Inputs:** None
- **Outputs/Return:** Pointer to a newly allocated `mutex_t`; ownership transferred to caller.
- **Side effects:** Allocates memory; creates an OS-level synchronization primitive.
- **Calls:** Not inferable from this file (defined in `common/mutex.c`).
- **Notes:** Caller is responsible for the lifetime of the returned handle; no corresponding `MutexFree` is declared here.

### MutexLock
- **Signature:** `void MutexLock(mutex_t *m)`
- **Purpose:** Acquires the mutex, blocking if it is already held by another thread.
- **Inputs:** `m` — pointer to an initialized `mutex_t` handle.
- **Outputs/Return:** `void`
- **Side effects:** Blocks the calling thread until the lock is acquired; modifies OS-level mutex state.
- **Calls:** Not inferable from this file.
- **Notes:** Behavior is undefined if `m` is `NULL` or uninitialized.

### MutexUnlock
- **Signature:** `void MutexUnlock(mutex_t *m)`
- **Purpose:** Releases a previously acquired mutex, allowing other waiting threads to proceed.
- **Inputs:** `m` — pointer to a locked `mutex_t` handle.
- **Outputs/Return:** `void`
- **Side effects:** Releases OS-level mutex; may unblock another thread.
- **Calls:** Not inferable from this file.
- **Notes:** Must only be called by the thread that currently holds the lock.

## Control Flow Notes
This header is a shared utility consumed by multi-threaded tool code (e.g., `common/threads.c`, `code/bspc/`, `q3map/`). It is not part of the game runtime frame loop; it is used during offline BSP compilation and AAS generation where worker threads need to guard shared data structures.

## External Dependencies
- No includes in this header.
- `MutexLock`, `MutexUnlock`, `MutexAlloc` — defined elsewhere (expected in `common/mutex.c`), with the actual platform implementation (pthreads, Win32 `CRITICAL_SECTION`, etc.) hidden behind the `void *` abstraction.
