# code/game/g_public.h — Enhanced Analysis

## Architectural Role

This file defines the critical **binary ABI contract** between the authoritative server engine (`code/server/`) and the pluggable game logic module (QVM or DLL). It establishes three key boundaries: (1) the entity data layout the server reads directly from game memory for collision and snapshot visibility, (2) the entry-point table for frame-driven game simulation and client lifecycle callbacks, and (3) the complete syscall enumeration for engine services the game module invokes (including bot AI tunneling). The `GAME_API_VERSION` versioning scheme allows game DLLs to target specific engine versions without full recompilation of the server.

## Key Cross-References

### Incoming (who depends on this file)

- **Server engine** (`code/server/sv_game.c`, `sv_snapshot.c`, `sv_world.c`):
  - Reads `sharedEntity_t` memory layout directly via pointer arithmetic after calling `G_LOCATE_GAME_DATA` — no function call overhead per entity, critical for handling 1024+ entities per frame
  - Interprets `svFlags` bitfield to determine whether to send entity to clients (`SVF_NOCLIENT`, `SVF_BROADCAST`, `SVF_SINGLECLIENT`)
  - Uses `entityShared_t.currentOrigin/Angles` and collision bounds for sector-tree linking and swept-box traces
  - Dispatches `gameExport_t` enum indices to the game VM's `vmMain` export

- **Game module** (`code/game/*.c`):
  - All files implement or call through the syscalls defined in `gameImport_t` (the trap_* wrappers)
  - Top-level `vmMain` function (in `g_main.c`) receives and dispatches `gameExport_t` opcodes from the server each frame

### Outgoing (what this file depends on)

- **Shared data types** (`code/game/q_shared.h`, `bg_public.h`):
  - `entityState_t` — what the network sees (spawned/despawned entities, position, animation state)
  - `playerState_t` — client's authoritative physics state
  - `usercmd_t` — client input commands
  - `vec3_t`, `qboolean`, `vmCvar_t` — foundational types
  - `trace_t` — collision query results from engine

- **BotLib subsystem** (via enum range 200–599):
  - This file does NOT include botlib headers; instead it reserves a range of syscall opcodes for botlib routines
  - The server's `SV_BotLibSystemCalls` dispatcher translates these opcodes to botlib function calls
  - This **decouples botlib from the game module's binary dependency**: botlib can be updated or even replaced without relinking the game DLL

## Design Patterns & Rationale

1. **Direct-Memory-Read ABI**: Rather than wrapping every entity access in a syscall, the server calls `G_LOCATE_GAME_DATA(gEnts, numGEntities, sizeofGEntity_t, ...)` once at init, then reads entity state directly. This trades sandbox isolation for per-frame performance (tracing/linking 1000+ entities would be prohibitively slow if every field access were a syscall).

2. **Bitfield Entity Flags** (the SVF_* constants): A low-level component system predating modern ECS. Each bit controls a distinct server behavior (`NOCLIENT`, `BROADCAST`, `SINGLECLIENT`, `CAPSULE`, etc.), allowing the server to interpret entity visibility/collision without consulting game logic.

3. **Syscall Enumeration Versioning**: The `gameImport_t` enum is ordered and versioned—new syscalls are appended to maintain binary compatibility with older game DLLs. The server dispatcher (`SV_GameSystemCalls`) looks up the enum value and invokes the corresponding engine function.

4. **Lazy Linking**: BotLib is not linked to the game module; instead, the game calls syscall opcodes in the 200–599 range, which the server tunnels to the botlib library. This allows the game to use AI services without carrying a botlib dependency, and allows botlib upgrades without game DLL recompilation.

## Data Flow Through This File

1. **Startup**:
   - Server calls `GAME_INIT` → game spawns all entities, calls `G_LOCATE_GAME_DATA` to register where entities live in game memory
   - Server obtains direct pointer to `gentity_t[]` array cast as `sharedEntity_t[]`

2. **Per-Frame**:
   - Server calls `GAME_RUN_FRAME` → game updates all entity think functions, physics, collision
   - Server reads `sharedEntity_t.r.currentOrigin/Angles` and `svFlags` directly (no syscall) to determine visibility/collision
   - Game calls `G_LINKENTITY` / `G_UNLINKENTITY` (syscalls) when entity bounds change; server updates sector tree internally

3. **Snapshot Building** (`sv_snapshot.c`):
   - Server enumerates entities, checks PVS visibility, then checks `SVF_NOCLIENT` / `SVF_SINGLECLIENT` flags
   - Delta-encodes `entityState_t s` and sends to clients via network

4. **Bot AI Integration**:
   - Game calls `trap_BotLib*` syscalls (opcodes 200–599)
   - Server's `SV_BotLibSystemCalls` converts opcode → botlib function pointer → botlib returns result

## Learning Notes

- **Why entities are split** into `entityState_t s` + `entityShared_t r`: network-transmitted state vs. authoritative server-only state. The split clarifies what is sent to clients and what stays server-side.
  
- **Idiomatic Q3 pattern**: Direct memory access is a common engine optimization in late-90s game code (Half-Life's edict_t model used the same approach). Modern engines use component queries (ECS) to avoid this tight coupling, but Q3's design reflects performance constraints of its era.

- **Capsule collision flag** (`SVF_CAPSULE`): Added post-release; indicates the collision system should use an axis-aligned capsule instead of a bounding box, improving movement smoothness. The flag allows per-entity collision shape selection without adding a syscall.

- **Portal visibility** (`SVF_PORTAL`): The server merges two PVS clusters (from `origin` and `origin2`) when this flag is set, allowing entities that span portal boundaries to be correctly visible on both sides.

## Potential Issues

- **Direct memory access assumption**: If the server and game DLL are ever built with mismatched struct packing or alignment, `sizeof(gentity_t)` will differ and entity array iteration will read garbage. Q3 mitigates this via build-time assertions and matching compiler flags, but it's fragile.

- **No opcode validation**: The `gameImport_t` syscall dispatcher in `SV_GameSystemCalls` doesn't bounds-check the opcode; an invalid syscall ID could index out of bounds. Q3 relies on game DLL correctness.
