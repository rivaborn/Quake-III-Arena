# code/game/g_mem.c — Enhanced Analysis

## Architectural Role

`g_mem.c` provides a **private, fixed-budget memory allocator** exclusively for the server-side game logic VM (`code/game`). It operates as a completely isolated memory island within the engine: the game module does not call into the engine's zone or hunk allocators (via `trap_*` syscalls), and the engine never manages or introspects this pool. This separation enforces a critical architectural boundary—the QVM game module is a **stateful, sandboxed sandbox** that owns its entire per-map state lifecycle. At map load, `G_InitMemory` reclaims the 256 KB pool; at map shutdown, all allocations are silently discarded alongside the entity simulation.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/g_spawn.c`**: `G_NewString` duplicates entity key-value strings into the pool via `G_Alloc`
- **`code/game/g_main.c`**: `G_InitGame` calls `G_InitMemory` during `GAME_INIT` phase to reset the pool before entity spawning
- **`code/game/g_*.c` (all files)**: Any function allocating entity-specific state structures, projectiles, temporary state
- **`code/game/ai_*.c`** (bot AI): allocate bot state, goal structures, decision trees
- **Server console**: `Svcmd_GameMem_f` is registered as a server command (via `code/game/g_svcmds.c`) to expose memory status

### Outgoing (what this file depends on)
- **`code/game/g_main.c`**: `G_Printf` (wraps `trap_Printf`), `G_Error` (wraps `trap_Error`, non-returning)
- **`code/game/g_local.h`**: Brings in `q_shared.h`, `bg_public.h`, `g_public.h` for basic types and macros
- **`g_debugAlloc` cvar**: Defined in `g_main.c`, consumed for conditional debug logging

## Design Patterns & Rationale

### Bump-Pointer Arena Allocator
The allocator is **stateless**: it maintains only a single offset (`allocPoint`) and hands out monotonically increasing addresses. This is the **opposite of a free-list or buddy allocator**—there is no free operation, no coalescing, no fragmentation tracking. 

**Rationale for this design in the game module context:**
- **Entity lifetime dominance**: Server entities are mostly spawned at level load or during gameplay as projectiles/pickups; they persist until destroyed, then the map ends and the entire pool is reclaimed. Few allocations are ever freed mid-map.
- **Zero-overhead allocation**: A pointer increment is O(1) and allocation-fast. No metadata overhead, no free-list traversal.
- **Sandbox containment**: A fixed 256 KB pool prevents the game VM from starving the engine with unbounded allocation. If the limit is hit, it's a fatal design error (too many entities), not a graceful degradation—which is appropriate for a 2005-era engine.
- **Contrast with engine allocators**: The engine's zone allocator (`qcommon`) is persistent and can fragment across the entire game session. The game module's pool is ephemeral and reclaimed at map shutdown, so fragmentation is irrelevant.

### 32-Byte Alignment Mask
`allocPoint += ( size + 31 ) & ~31` rounds every allocation to the next 32-byte boundary. This is **conservative for 2005 hardware** (L1 cache lines were typically 64 bytes on x86, 128 on PowerPC), but it's a safe choice that prevents unaligned access penalties across all targets (x86, PPC, ARM—the three platforms Q3 was ported to).

## Data Flow Through This File

1. **Map Load → Initialization:**
   - Engine calls `vmMain(GAME_INIT, ...)` → game module bootstrap
   - `G_InitGame` → `G_InitMemory` sets `allocPoint = 0`, pool is now ready

2. **Entity Spawn Phase:**
   - `G_SpawnEntity` → `G_NewString` → `G_Alloc` for entity-specific strings/state
   - Bot AI setup → allocates `bot_state_t`, goal tables, etc. via repeated `G_Alloc`
   - Entity class constructors allocate per-entity data (player state, projectile timers, mover curves)

3. **Runtime:**
   - Throughout the map session, allocations continue (temporary structures, combat events, item pickups)
   - `G_Printf` logs allocation size if `g_debugAlloc` is set (slow path for debugging)
   - `Svcmd_GameMem_f` reports current usage to server console

4. **Map Shutdown:**
   - No cleanup needed: static pool buffer is abandoned, `allocPoint` is left wherever it ended
   - At next map load, `G_InitMemory` resets `allocPoint = 0` and the pool is reused
   - **Silent aliasing**: Any stale pointers from the prior map become dangling; code must not retain them across level boundaries

## Learning Notes

**Idiomatic to this era (2005):**
- **No RAII, no constructors/destructors**: Allocation and initialization are separate. There is no notion of "allocated ↔ constructed" lifecycle; you allocate a block and then manually initialize its contents.
- **Global mutable state as a feature, not a bug**: `allocPoint` and `memoryPool` are file-static globals. No encapsulation. This is typical for C code of that era.
- **Fixed budgets as acceptable failure modes**: Hitting the 256 KB limit is a fatal error (`G_Error`). Modern engines would use dynamic allocation or streaming; Q3 says "fit in 256 KB or you're doing it wrong."

**Modern equivalents:**
- **Linear/arena allocators** are experiencing a resurgence in modern game engines (Gears 5, recent Rust game libraries) for performance-critical paths. The pattern is timeless.
- **VM sandboxing**: Isolating the game module's memory is a precursor to modern plugin/mod systems. Modern engines (Unreal with plugins, Godot with gdscript) still use similar sandbox isolation for untrusted code.

**Connections to engine concepts:**
- This is a **lifetime-based allocator** (Rust-era terminology): the entire pool's lifetime is the duration of one map. No explicit free; implicit reclaim at scope exit (map end).
- The **pool reset** is a form of **generation-based memory reclamation**, similar to generational GC (bump the generation counter, invalidate old pointers).

## Potential Issues

1. **Signed integer overflow in size check:**
   ```c
   if ( allocPoint + size > POOLSIZE ) { ... }
   ```
   If `size` is negative (e.g., via a signed-to-unsigned type confusion in caller), the addition could wrap, bypassing the check. In practice, this is unlikely because callers compute sizes with `sizeof()` or `strlen() + 1`, both of which are non-negative. However, a defensive fix would be `if (size < 0 || allocPoint + size > POOLSIZE)`.

2. **Pool exhaustion is fatal:**
   `G_Error` is non-returning; it does a `longjmp` back to the engine's `Com_Error` handler. This kills the entire map session. Modern engines would either grow the pool dynamically or provide graceful degradation (defer non-critical allocations). For Q3, this is acceptable: a single map's entity count is known at design time, and 256 KB was deemed sufficient in 2005.

3. **Silent aliasing after reset:**
   The first-pass mentions this. If any code (e.g., a buggy cvar callback or a lingering pointer in a global struct) retains an address across `G_InitMemory`, it will crash when dereferenced. The burden is entirely on callers to not do this. No mechanism prevents it.
