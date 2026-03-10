# code/game/g_local.h — Enhanced Analysis

## Architectural Role

This file serves as the internal API hub for the entire game module, acting as the sole point of integration between the game VM and the engine. Every game subsystem (combat, movement, items, team/CTF, bot AI, entity simulation) passes through this header's declarations and structures. The file enforces the ABI contract that `gentity_t`'s first two fields and `gclient_t`'s first field must match the engine's expected layout so `trap_LocateGameData` can expose them directly to the server's snapshot/trace pipelines without copying.

## Key Cross-References

### Incoming (who depends on this file)
- **All game module `.c` files** include this header unconditionally
- **Engine (`code/server/sv_game.c`)** calls `vmMain` with `GAME_INIT`, `GAME_RUN_FRAME`, `GAME_CLIENT_THINK`, and `GAME_SHUTDOWN`; expects `level` and `g_entities` to be registered and valid per `trap_LocateGameData` contract
- **Renderer and client** access replicated state via `gentity_t.s` (entityState_t) and `gclient_t.ps` (playerState_t), which are read-only copies
- **BotLib** is never directly linked; all AI queries route through `trap_BotLib*` syscall opcodes (200–599) declared here

### Outgoing (what this file depends on)
- **`q_shared.h`**: base types (`qboolean`, `vec3_t`, `vec4_t`, `int`, etc.), `entityState_t`, `playerState_t`, `usercmd_t`
- **`bg_public.h`**: shared game constants (`team_t`, `gametype_t`, weapon enums, `gitem_t`, `trace_t`, pmove input/output)
- **`g_public.h`**: engine API contract (`sharedEntity_t`, `entityShared_t`, game module import/export, syscall opcode enums)
- **`g_team.h`**: CTF and team gameplay function prototypes

## Design Patterns & Rationale

**1. Opaque Entity Pooling**
- `gentity_t` is a fixed, maximally-sized structure holding every possible entity attribute (position, physics, callbacks, health, damage, sounds, etc.)
- Avoids dynamic type dispatch or virtual methods; callbacks (`think`, `touch`, `use`, `die`, `pain`) are function pointers set per-entity at spawn time
- Rationale: efficient cache locality, predictable memory layout for the engine to access, minimal indirection cost in hot loops

**2. State Machine Enums**
- `moverState_t`, `clientConnected_t`, `spectatorState_t`, `playerTeamStateState_t` encode FSM states as integers
- Enables tight switch-based dispatch in `think` callbacks without polymorphism overhead
- Rationale: compile-era C pragmatism; enum FSMs are idiomatic to id-era engines (Doom, Quake, HL1)

**3. Persistent Layer Separation**
- `clientPersistant_t` survives respawns within a level; cleared only on level/team change
- `clientSession_t` survives level transitions and server restarts via cvar serialization
- Rationale: supports tournament mode (win/loss tracking across maps) and respawn-friendly item tracking (`initialSpawn`, `predictItemPickup`)

**4. Callback Closure Encoding**
- Function pointers (`think`, `pain`, `die`, `reached`, `blocked`, `touch`, `use`) implicitly capture state via `self` pointer
- All closures are closed over `gentity_t` (the entity itself) or `gclient_t` (the client)
- Rationale: no heap-allocated closures; predictable memory ownership; compatible with QVM bytecode (no closures in QVM)

## Data Flow Through This File

**Entity Lifecycle:**
```
G_Spawn() → G_InitGentity() → [spawn functions set callbacks/state] 
→ [every frame: gentity_t.think(self) called by game loop] 
→ G_FreeEntity() → unlink from world → mark as free
```

**Client Lifecycle:**
```
Connect → Session restored from cvars via G_ReadSessionData() 
→ Spawn at spawn point → [every frame: player think/predict] 
→ Respawn (ps/pers cleared, sess preserved) OR Disconnect (sess saved to cvars)
```

**Server→Client Replication:**
```
gentity_t.s (entityState_t) + gclient_t.ps (playerState_t) 
→ delta-encoded by server 
→ received by client 
→ cgame consumes via `trap_GetSnapshot()` 
→ client-side prediction via identical `bg_pmove` code
```

## Learning Notes

- **Hub Header Pattern:** `g_local.h` exemplifies the "hub header" idiom in large C systems. All per-module function declarations, global extern definitions, and constants live here to minimize include chains and enforce a single module API surface.
- **Opaque Handle Idiom:** The `gentity_t` index (0–MAX_GENTITIES−1) acts as an opaque integer handle. The engine stores this handle in `entityState_t.number`, allowing the server to cross-reference entities by ID without pointer serialization.
- **Determinism via Shared Code:** `bg_pmove.c` is compiled identically into both `game` and `cgame` VMs so client prediction matches server physics exactly (up to floating-point determinism limits). This is a core architectural constraint captured in comments like `// DO NOT MODIFY ANYTHING ABOVE THIS, THE SERVER EXPECTS THE FIELDS IN THAT ORDER!`
- **Idiomatic Callback Style:** Entity behavior is defined via function pointers rather than virtual methods or message dispatch. This is typical of entity systems from the id Software era (Doom, Quake) and remains efficient for simple game logic.

## Potential Issues

1. **Monolithic Structure:** `gentity_t` is a 500+ byte structure with many unused fields per entity type. Specialized entity structs or tagged unions might reduce memory waste, but would require type-specific callback dispatch.
2. **Syscall Opcode Fragility:** The 600+ `trap_*` syscall opcodes are tightly coupled to engine opcode assignments. Adding/removing syscalls requires coordinated engine/game changes; versioning is minimal (only `vmMain` has a `VM_VERSION` contract).
3. **Global State Coupling:** All game state lives in the global `level` singleton and `g_entities[]` array. Changes to `level_locals_t` or `gentity_t` layout require engine rebuild, making refactoring risky.
4. **Missing Bounds Documentation:** Some fields lack explicit size limits (e.g., `soundPos1`, `sound1to2` are sound indices with no documented max; `clipmask` is a bitmask with no enumerated values).
