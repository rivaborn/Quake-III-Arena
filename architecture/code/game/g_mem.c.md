# code/game/g_mem.c

## File Purpose
Provides a simple bump-pointer memory allocator backed by a fixed 256 KB static pool for the game module. All allocations are permanent for the duration of a map session; there is no free operation.

## Core Responsibilities
- Allocate memory from a fixed-size static pool with 32-byte alignment
- Detect pool exhaustion and fatal-error on overflow
- Reset the pool at map/session start via `G_InitMemory`
- Expose current pool usage via a server console command

## Key Types / Data Structures
None.

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `memoryPool` | `char[262144]` | static (file) | The 256 KB backing buffer for all game allocations |
| `allocPoint` | `int` | static (file) | Bump pointer offset; marks the next free byte in the pool |

## Key Functions / Methods

### G_Alloc
- **Signature:** `void *G_Alloc( int size )`
- **Purpose:** Allocates `size` bytes from the static pool, 32-byte aligned.
- **Inputs:** `size` — requested allocation size in bytes (signed int).
- **Outputs/Return:** Pointer to the allocated region; `NULL` is unreachable in practice (error fires first).
- **Side effects:** Advances `allocPoint` by `(size + 31) & ~31`. Calls `G_Printf` if `g_debugAlloc` is set. Calls `G_Error` (fatal) on overflow.
- **Calls:** `G_Printf`, `G_Error`
- **Notes:** The alignment mask rounds up to the next 32-byte boundary, preventing unaligned access. The `NULL` return after `G_Error` is dead code — `G_Error` does not return. No thread safety; game module is single-threaded.

### G_InitMemory
- **Signature:** `void G_InitMemory( void )`
- **Purpose:** Resets the pool by zeroing `allocPoint`, effectively reclaiming all prior allocations.
- **Inputs:** None.
- **Outputs/Return:** `void`
- **Side effects:** Sets `allocPoint = 0`; the pool buffer itself is not zeroed.
- **Calls:** None.
- **Notes:** Called at map start (via `G_InitGame` in `g_main.c`). Prior pointers handed out by `G_Alloc` become silently aliased after a reset — callers must not retain them across level changes.

### Svcmd_GameMem_f
- **Signature:** `void Svcmd_GameMem_f( void )`
- **Purpose:** Server console command handler that prints current pool usage.
- **Inputs:** None (reads `allocPoint` and `POOLSIZE` directly).
- **Outputs/Return:** `void`
- **Side effects:** Calls `G_Printf` to emit a status line to the console.
- **Calls:** `G_Printf`
- **Notes:** Registered as a server command; invoked by `g_svcmds.c`/`ConsoleCommand`.

## Control Flow Notes
`G_InitMemory` is called during map initialization (init phase) to reset the pool before any entities or bot state are spawned. `G_Alloc` is called throughout the spawn phase by string duplication helpers (e.g., `G_NewString` in `g_spawn.c`) and bot/AI subsystems. There is no shutdown-phase involvement; the static array is simply abandoned at map end.

## External Dependencies
- **Includes:** `g_local.h` (pulls in `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `g_debugAlloc` — `vmCvar_t` extern declared in `g_local.h`, defined in `g_main.c`
  - `G_Printf` — defined in `g_main.c`; wraps `trap_Printf`
  - `G_Error` — defined in `g_main.c`; wraps `trap_Error` (non-returning)
