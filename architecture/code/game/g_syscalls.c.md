# code/game/g_syscalls.c

## File Purpose
Implements the DLL-side system call interface for the game module, providing typed C wrapper functions around a single variadic `syscall` function pointer set by the engine at load time. This file is excluded from QVM builds, where `g_syscalls.asm` is used instead.

## Core Responsibilities
- Receive and store the engine's syscall dispatch function pointer via `dllEntry`
- Wrap every engine API call (file I/O, cvars, networking, collision, etc.) as typed C functions
- Bridge float arguments through `PASSFLOAT` to avoid ABI issues with variadic integer-only syscall conventions
- Expose the full BotLib/AAS API surface to game logic via trap functions
- Provide entity action (EA) wrappers for bot input simulation

## Key Types / Data Structures
None defined in this file. Uses types from `g_local.h` (e.g., `gentity_t`, `vmCvar_t`, `trace_t`, `vec3_t`, `usercmd_t`, `pc_token_t`).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `syscall` | `static int (QDECL *)( int arg, ... )` | static | Function pointer to the engine's syscall dispatcher; initialized to `-1` (invalid), set by `dllEntry` |

## Key Functions / Methods

### dllEntry
- **Signature:** `void dllEntry( int (QDECL *syscallptr)( int arg, ... ) )`
- **Purpose:** Engine entry point called immediately after loading the game DLL; installs the syscall dispatch pointer.
- **Inputs:** `syscallptr` — engine-provided variadic syscall function.
- **Outputs/Return:** void
- **Side effects:** Writes to the file-static `syscall` pointer.
- **Calls:** None.
- **Notes:** Must be called before any `trap_*` function is invoked; calling traps before this dereferences address `-1`.

### PASSFLOAT
- **Signature:** `int PASSFLOAT( float x )`
- **Purpose:** Reinterprets a `float` bit-pattern as an `int` so it can be passed through the integer-typed variadic syscall without conversion loss.
- **Inputs:** `x` — float value.
- **Outputs/Return:** Bit-identical `int` representation.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Required for every `float` argument passed to `syscall`; forgetting it produces silent value corruption.

### trap_LocateGameData
- **Signature:** `void trap_LocateGameData( gentity_t *gEnts, int numGEntities, int sizeofGEntity_t, playerState_t *clients, int sizeofGClient )`
- **Purpose:** Registers the game's entity and client arrays with the server engine so the server can access game state directly.
- **Inputs:** Base pointers and element sizes for both arrays.
- **Outputs/Return:** void
- **Side effects:** Engine stores these pointers; miscalling crashes the server.
- **Calls:** `syscall( G_LOCATE_GAME_DATA, ... )`
- **Notes:** Called once during game init (`G_InitGame`).

### trap_Trace / trap_TraceCapsule
- **Signature:** `void trap_Trace( trace_t *results, const vec3_t start, const vec3_t mins, const vec3_t maxs, const vec3_t end, int passEntityNum, int contentmask )`
- **Purpose:** Perform a box/capsule trace through the collision world via the engine.
- **Side effects:** Fills `*results`.
- **Calls:** `syscall( G_TRACE / G_TRACECAPSULE, ... )`

### trap_AAS_Time
- **Signature:** `float trap_AAS_Time(void)`
- **Purpose:** Retrieves the current AAS simulation time as a float.
- **Outputs/Return:** Float decoded via pointer-cast from the int returned by `syscall`.
- **Notes:** Uses the same bit-reinterpret pattern as `PASSFLOAT` in reverse; parallels `trap_Characteristic_Float` and `trap_BotAvoidGoalTime`.

### Notes (trivial wrappers)
- All remaining `trap_*` functions are single-line forwarding wrappers: marshal arguments → `syscall(ENUM_CONSTANT, ...)` → return result. No logic beyond float promotion via `PASSFLOAT` where needed.

## Control Flow Notes
`dllEntry` is called by the engine during DLL load (before any game functions). After that, all game-side code calls `trap_*` functions freely throughout init, per-frame, and shutdown phases. This file has no frame loop of its own — it is purely a call-forwarding layer invoked on demand by the rest of `code/game/`.

## External Dependencies
- `code/game/g_local.h` — pulls in `q_shared.h`, `bg_public.h`, `g_public.h`, and all game type/enum definitions
- `G_PRINT`, `G_ERROR`, `G_LOCATE_GAME_DATA`, `BOTLIB_*`, `G_TRACE`, etc. — syscall opcode enumerations defined in `g_public.h` / `botlib.h` (defined elsewhere)
- `gentity_t`, `playerState_t`, `trace_t`, `vmCvar_t`, `usercmd_t`, `pc_token_t` — defined elsewhere
- `QDECL` — calling convention macro, defined in `q_shared.h`
