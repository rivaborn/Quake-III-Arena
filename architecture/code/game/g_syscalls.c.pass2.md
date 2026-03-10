# code/game/g_syscalls.c — Enhanced Analysis

## Architectural Role
This file serves as the **ABI bridge** between the game module (DLL) and the Quake 3 engine core. It implements the game-side half of the syscall dispatch mechanism, converting all engine API requests from typed C function calls into flat integer opcode + variadic argument tuples forwarded through a single function pointer. This design enabled Q3 to support both native DLL and QVM bytecode game modules without code duplication, as the same `trap_*` interface is called by game logic regardless of backend.

## Key Cross-References

### Incoming (who depends on this file)
- **Every file in `code/game/`**: All game logic modules call `trap_*` functions for engine services:
  - `g_main.c`, `g_active.c`, `g_client.c`, `g_combat.c` → collision queries, entity linking, configstring updates
  - `g_bot.c`, `ai_main.c`, `ai_dmq3.c` → `trap_BotLib*` range (200–599) for AAS navigation, movement simulation, goal selection
  - `bg_pmove.c` (shared with cgame) → `trap_Trace`, `trap_PointContents` for authoritative movement physics
- **Game module initialization** (`vmMain` entry): Must call `dllEntry` before any other syscall
- **Implicit callers via includes**: Every `.c` file in `code/game/` includes `g_local.h`, which declares these function prototypes

### Outgoing (what this file depends on)
- **Engine-provided `syscall` function pointer** (address `-1` initially, set by `dllEntry`):
  - Maps to `SV_GameSystemCalls` in the server (via `sv_game.c:SV_Call`, or directly by VM dispatcher)
  - Each `trap_*` call marshals arguments and forwards to this dispatcher
  - Syscall opcodes (e.g., `G_TRACE`, `BOTLIB_AAS_TIME`) defined in `code/game/g_public.h` and `code/botlib/botlib.h`
- **Global state**: Modifies `syscall` static variable once at module load
- **Type definitions from `g_local.h`**: All entity, client, trace, and BotLib structures

## Design Patterns & Rationale

**1. ABI Bridge (Syscall Dispatch)**
- The engine provides a single variadic function pointer at runtime; `g_syscalls.c` translates all game-engine interactions into syscall dispatch calls
- Enables **identical game source code** to compile as either native DLL (using this file) or QVM bytecode (using `g_syscalls.asm` assembler stubs instead)
- **Trade-off**: No compile-time type safety on syscall arguments; all type checking done manually via opcode enum definitions

**2. Float Bit-Reinterpretation via `PASSFLOAT`**
- Q3's variadic syscall signature is `int (QDECL *)( int arg, ... )` — all arguments are integers
- Floats cannot be passed losslessly through a variadic int-only interface; `PASSFLOAT` reinterprets IEEE 754 bit patterns as ints without conversion loss
- **Reverse pattern**: Functions like `trap_AAS_Time()` receive an `int` from syscall, then cast it back via `(*(float*)&temp)`
- This is technically undefined behavior in strict C, but relies on the common assumption that IEEE float layout == 32-bit bitwise representation

**3. Lazy Initialization**
- `syscall` is initialized to `-1` (invalid pointer); the first dereference would crash
- `dllEntry` **must** be called by the engine immediately after loading the DLL, before any game code runs
- Calling any `trap_*` before `dllEntry` has executed will crash with a null-pointer (or address `-1`) dereference

## Data Flow Through This File

```
Game Module (vmMain entry)
  ↓
  dllEntry( syscallptr )
    → Store syscallptr in static 'syscall'
  ↓
Game Logic (g_main.c, g_bot.c, etc.)
  ↓
  trap_* wrapper functions
    → Marshal typed args → Pack as variadic int args
    → Call syscall(OPCODE, arg1, arg2, ...)
    → Receive int result
    → Reinterpret result (if float) or return as-is
  ↓
Engine's SV_GameSystemCalls (via sv_game.c)
  → Access qcommon services: CM_Trace, CM_LinkEntity, cvar_t, configstrings, etc.
  → Access server data: client states, entity arrays, userinfo, bot AI
  ↓
Result flows back to game module
```

**Key state transitions**:
- Uninitialized → `dllEntry` called → `syscall` valid → game logic can run
- Entity queries flow **in** (from engine): `trap_EntitiesInBox`, `trap_Trace`, `trap_PointContents`
- Entity updates flow **out**: `trap_SetConfigstring`, `trap_LinkEntity`, `trap_SendServerCommand`
- Bot AI flows **out** via `trap_BotLib*` range; AAS/movement state flows **back in** via results

## Learning Notes

**Q3-Specific Patterns**:
1. **DLL Injection vs. EXE Linking**: Modern game engines link game logic statically; Q3 loaded it dynamically as a DLL, requiring a syscall bridge. This enabled swappable game modules without engine recompilation.
2. **Syscall Dispatch as Language Boundary**: The syscall acts as a mini-RPC layer. Each opcode enum is a versioned API; old DLLs with old opcode constants were rejected to prevent misalignment.
3. **Shared Code (bg_pmove.c)**: Physics is compiled identically into game and cgame VMs; `bg_pmove.c` calls `trap_Trace` agnostically, with the engine routing to either server-side traces or client-side traces depending on caller context.
4. **BotLib as Coprocess**: The `trap_BotLib*` opcode range (200–599) is large, indicating botlib is a self-contained subsystem called through syscall rather than linked. This isolation enabled iterative bot AI improvements post-launch.

**Contrast with Modern Engines**:
- Unreal/Unity: Game code linked directly, full C++ ABI compatibility
- Q3: Syscall-mediated, no direct symbol linkage, opcode versioning
- This trade-off prioritized **flexibility** (swap game DLL, enforce sandbox) over **performance** (no call overhead, direct symbol access)

## Potential Issues

1. **Initialization Order Violation**: If any game code runs before `dllEntry` is called, dereferencing the `-1` pointer is undefined behavior (may appear to work on some platforms, crash on others).
2. **Float Reinterpretation UB**: Casting pointers between `int*` and `float*` violates strict-aliasing rules in C99+; relies on compiler pragmatism and common ABI assumptions.
3. **No Type Checking at Syscall Boundary**: Passing wrong argument count or types to `syscall(OPCODE, ...)` is not caught at compile time. Misalignment between game/engine opcode definitions can silently corrupt arguments.
4. **Limited Extensibility**: Adding new syscalls requires coordinating opcode numbers across game, cgame, ui, and all helper modules; accidental opcode collisions are a latent risk in large refactors.
