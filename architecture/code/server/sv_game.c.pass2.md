# code/server/sv_game.c — Enhanced Analysis

## Architectural Role

`sv_game.c` is the **VM boundary membrane** of the server subsystem — the single file through which all game logic crosses from the sandboxed QVM into the engine. It sits at the intersection of four major subsystems: the server frame loop (`sv_main.c`, `sv_init.c`), the collision world (`qcommon/cm_*.c`), the world-space entity system (`sv_world.c`), and the bot library (`botlib/be_interface.c`). The game VM is entirely blind to all engine internals; every capability it needs — traces, entity linking, cvars, filesystem, bot AI — is funneled through the single `SV_GameSystemCalls` dispatch table registered at `VM_Create` time. This file is therefore the single point of failure, the single audit surface, and the single extension point for the entire game–engine contract.

---

## Key Cross-References

### Incoming (who depends on this file)

| Caller | What it uses |
|--------|-------------|
| `code/server/sv_main.c` | `SV_InitGameProgs`, `SV_ShutdownGameProgs`, `SV_RestartGameProgs`, `SV_GameCommand` — full VM lifecycle |
| `code/server/sv_snapshot.c` | `SV_GentityNum`, `SV_NumForGentity`, `SV_GameClientNum` — snapshot building iterates all entities |
| `code/server/sv_world.c` | `SV_SvEntityForGentity`, `SV_GEntityForSvEntity` — sector-tree link/unlink and trace hit resolution |
| `code/server/sv_client.c` | `SV_GameSendServerCommand`, `SV_GameDropClient` indirectly via the server command layer |
| `code/game/g_syscalls.c` (game VM) | All `trap_*` wrappers compile to indexed syscall stubs that resolve to entries in `SV_GameSystemCalls` |
| `code/game/g_bot.c` | `BOTLIB_*` opcodes forwarded through `SV_GameSystemCalls` to `botlib_export` |
| Global `botlib_export` | Read by `SV_BotLibSetup` (in `sv_bot.c`) and written here; all botlib dispatch paths in this file consume it |

### Outgoing (what this file depends on)

| Subsystem | Symbols used |
|-----------|-------------|
| **VM host** (`qcommon/vm.c`) | `VM_Create`, `VM_Call`, `VM_Free`, `VM_Restart`, `VM_ArgPtr` — the entire VM lifecycle |
| **Collision** (`qcommon/cm_*.c`) | `CM_PointLeafnum`, `CM_LeafCluster`, `CM_ClusterPVS`, `CM_LeafArea`, `CM_AreasConnected`, `CM_AdjustAreaPortalState`, `CM_InlineModel`, `CM_ModelBounds`, `CM_TransformedBoxTrace`, `CM_EntityString` |
| **Server world** (`sv_world.c`) | `SV_LinkEntity`, `SV_UnlinkEntity`, `SV_AreaEntities`, `SV_Trace`, `SV_PointContents`, `SV_ClipHandleForEntity` |
| **Server bot** (`sv_bot.c`) | `SV_BotAllocateClient`, `SV_BotFreeClient`, `SV_BotLibSetup`, `SV_BotLibShutdown`, `SV_BotGetSnapshotEntity`, `SV_BotGetConsoleMessage` |
| **Server client** (`sv_client.c`) | `SV_DropClient`, `SV_SendServerCommand`, `SV_ClientThink` |
| **Bot library** (`botlib/be_interface.c`) | All `botlib_export->aas.*`, `botlib_export->ea.*`, `botlib_export->ai.*` calls — ~50+ function pointer dereferences |
| **qcommon services** | `FS_*`, `Cvar_*`, `Cmd_*`, `Cbuf_ExecuteText`, `COM_Parse`, `Com_RealTime`, `Com_Error`, `Sys_Milliseconds`, `Sys_SnapVector`, `MatrixMultiply`, `AngleVectors`, `PerpendicularVector` |
| **Server globals** (`server.h`) | `sv.gentities`, `sv.gentitySize`, `sv.gameClients`, `sv.gameClientSize`, `sv.svEntities`, `sv.entityParsePoint`, `svs.clients`, `gvm`, `sv_maxclients` |

---

## Design Patterns & Rationale

**Syscall dispatch table (opcode switch):** `SV_GameSystemCalls` is a single `switch` over integer opcodes. This is the standard Quake III VM ABI — it trades runtime type safety for a minimal, version-stable, platform-portable interface. The game module compiles against only the opcode constants in `g_public.h`; the engine can evolve implementation freely as long as it honors those opcodes. This is the same pattern as the `TRAP_*` ABI in cgame and ui — all three VMs use the same dispatch idiom via their respective syscall handlers.

**Variable-stride entity array:** `SV_GentityNum` / `SV_NumForGentity` use explicit byte-stride arithmetic (`sv.gentitySize`) rather than array indexing. This is because the game VM allocates `gentity_t` structs with game-private fields appended after the shared `sharedEntity_t` prefix. The server only ever sees the shared prefix; the game sees the full struct. This avoids a compile-time dependency on the game's private layout — the actual size is negotiated at runtime via `G_LOCATE_GAME_DATA`.

**`botlib_export` as a vtable:** Rather than linking botlib symbols directly, the engine acquires `botlib_export_t *` at init time (via `GetBotLibAPI`) and dispatches all bot calls through function pointers. This allows botlib to be a separately loaded DLL and insulates both sides from each other's ABI details.

**PVS variants:** Two nearly-identical PVS tests exist (`SV_inPVS` vs `SV_inPVSIgnorePortals`) because area-portal connectivity (door-blocks-sight) is a separate concept from cluster-based PVS. Game code needs both: AI pathfinding may want pure geometry visibility, while snapshot culling must respect dynamic portal state.

**`FloatAsInt` union trick:** Returning floats from a function typed `int (*)()` via union type-punning is a pre-C99-era workaround. Modern code would use `memcpy` or return a proper struct, but the VM ABI is fixed at `int`.

---

## Data Flow Through This File

```
Game VM (QVM bytecode / native DLL)
    │  trap_* stubs → syscall opcode in args[0]
    ▼
SV_GameSystemCalls(args[])
    ├─ VMA(x) = VM_ArgPtr(args[x])   ← untrusted VM pointers → engine-safe pointers
    ├─ VMF(x) = float cast of args[x]
    │
    ├─ G_LOCATE_GAME_DATA ──→ sv.gentities, sv.gameClients  (game → server shared state)
    ├─ G_LINKENTITY ─────────→ SV_LinkEntity → sv_world sector tree
    ├─ G_TRACE ──────────────→ SV_Trace → CM_* BSP sweep  → trace_t result in VM mem
    ├─ G_IN_PVS ─────────────→ SV_inPVS → CM_ClusterPVS → qboolean
    ├─ BOTLIB_* ─────────────→ botlib_export->aas/ea/ai.*
    ├─ G_SET_CONFIGSTRING ───→ SV_SetConfigstring → sv.configstrings (replicated to clients)
    └─ G_SEND_SERVER_COMMAND → SV_SendServerCommand → Netchan outbound queue

SV_InitGameVM (init path):
    CM_EntityString → sv.entityParsePoint  (entity string parse cursor)
    VM_Call(GAME_INIT) → game VM populates itself, calls G_LOCATE_GAME_DATA back
```

The key asymmetry: the server owns `sv.svEntities[]` (world-space partition data) but the game owns `sv.gentities[]` (logical entity state). This file bridges the two via `SV_SvEntityForGentity` / `SV_GEntityForSvEntity`.

---

## Learning Notes

- **The VM contract as an API surface:** The `G_*` and `BOTLIB_*` opcode sets are essentially a published API — any mod that compiles to QVM bytecode against these constants will run on any engine that implements them correctly. This is how Quake III modding works: mods are QVMs, not source patches.

- **`sv.entityParsePoint` as a cursor:** Entity parsing is stateful — the game calls `G_GET_ENTITY_TOKEN` repeatedly to walk the BSP entity string. The parse cursor is stored in `sv.entityParsePoint` (set in `SV_InitGameVM`). This is an idiom for streaming parsers without heap allocation.

- **No ECS here:** Q3's entity system is a flat array of fixed-slot structs with per-entity `think` function pointers — closer to an object array with vtable emulation than a modern ECS. The variable-stride trick (`gentitySize`) is an ad-hoc form of the "struct of arrays" / "components beyond base" pattern that modern engines formalize.

- **`BOTLIB_EA_ACTION` fall-through bug:** The first-pass noted that `BOTLIB_EA_ACTION` uses `break` instead of `return 0`, causing it to fall through to the default `return -1`. This is a real discrepancy — bot action commands return -1 instead of 0 to the game VM, though in practice the game side ignores the return value for action calls.

- **Hot-restart path (`SV_RestartGameProgs`):** `map_restart` preserves map geometry (no `CM_LoadMap`) but re-initializes game logic. The `VM_Restart` call reuses the loaded bytecode image without disk I/O. This is a non-obvious optimization: only the VM's memory is reset, not the BSP world.

---

## Potential Issues

- **No bounds validation on `VMA()` pointers:** `VM_ArgPtr` applies a `dataMask` to sandboxed QVM pointers, but native DLL builds (`VMI_NATIVE`) skip masking entirely. A compromised or buggy game DLL can pass any host pointer as a `VMA()` argument.
- **`BOTLIB_EA_ACTION` returns -1:** As noted above, likely a latent bug with no observable effect since callers ignore the return.
- **`SV_SetBrushModel` FIXME:** The automatic `SV_LinkEntity` call at the end is flagged for removal — callers may not expect it to auto-link, leading to double-link if they call `SV_LinkEntity` themselves afterward.
- **`sv.num_entities` vs `MAX_GENTITIES`:** `SV_LocateGameData` writes `sv.num_entities` from the game's reported count, but `SV_SvEntityForGentity` bounds-checks against `MAX_GENTITIES`. If the game reports a count larger than `MAX_GENTITIES`, the bounds check would still pass for out-of-range indices below `MAX_GENTITIES`.
