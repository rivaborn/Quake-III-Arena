# code/qcommon/cvar.c — Enhanced Analysis

## Architectural Role

`cvar.c` is the **global configuration backbone** of the Q3A engine, sitting at the center of `qcommon` and serving every other subsystem without depending on any of them (beyond memory and console utilities). It is the single source of truth for all runtime-configurable state: renderer quality settings, network parameters, cheat flags, server/userinfo that flows over the wire, and the defaults written to `q3config.cfg`. Because every subsystem — renderer, server, client, all three VM types, and botlib — registers and reads cvars, any change to the cvar system's ABI or flag semantics ripples engine-wide. Notably, botlib deliberately avoids this system and uses its own `l_libvar.c` to remain self-contained and portable outside Q3A.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/qcommon/common.c`** — calls `Cvar_Init` during `Com_Init`; polls `cvar_modifiedFlags` each frame to detect class-level changes (USERINFO, SERVERINFO, SYSTEMINFO bits).
- **`code/server/sv_main.c`, `sv_init.c`, `sv_client.c`** — call `Cvar_Get`/`Cvar_Set` directly for server-side state; call `Cvar_InfoString(CVAR_SERVERINFO)` to build configstring 0 sent to all clients.
- **`code/client/cl_main.c`, `cl_cgame.c`, `cl_ui.c`** — call `Cvar_Get`/`Cvar_Set`; `cl_cgame.c` and `cl_ui.c` dispatch VM syscalls `CG_CVAR_REGISTER`/`UI_CVAR_REGISTER` → `Cvar_Register`; `cl_main.c` calls `Cvar_InfoString(CVAR_USERINFO)` when building the `userinfo` configstring sent to the server.
- **`code/server/sv_game.c`** — routes game VM syscalls (GAME_CVAR_REGISTER, GAME_CVAR_UPDATE, GAME_CVAR_SET, etc.) to the cvar API; reads `cvar_cheats` indirectly via `Cvar_Set2`.
- **Renderer** — obtains `Cvar_*` pointers through the `refimport_t ri` vtable filled by `cl_main.c`; `tr_init.c` registers all `r_*` cvars at renderer startup.
- **`code/qcommon/cmd.c`** — `Cmd_ExecuteTokenizedString` calls `Cvar_Command` as a fallback when no registered command matches a token; this is how `r_fullscreen 0` typed at the console works.

### Outgoing (what this file depends on)

- **Zone allocator** (`Z_Free`, `CopyString` from `common.c`) — every cvar string (name, value, reset, latch) is a heap-allocated `CopyString`; the fixed-size pool stores only the struct itself.
- **Console/error** (`Com_Error`, `Com_Printf`, `Com_DPrintf`) — for fatal pool exhaustion, protection-flag messages, and debug output.
- **Command system** (`Cmd_Argc`, `Cmd_Argv`, `Cmd_AddCommand`) — `Cvar_Init` registers `toggle`, `set`, `sets`, `setu`, `seta`, `reset`, `cvarlist`, `cvar_restart`; console command handlers read tokens via `Cmd_Argv`.
- **Filesystem** (`FS_Printf`) — `Cvar_WriteVariables` writes `seta` lines for `CVAR_ARCHIVE` cvars.
- **Info-string utilities** (`Info_SetValueForKey`, `Info_SetValueForKey_Big`) — `Cvar_InfoString` uses these to build the key=value pairs sent over the network.

## Design Patterns & Rationale

- **Fixed-size pool + dual-structure indexing.** The 1024-entry `cvar_indexes[]` array eliminates fragmentation and makes `vmCvar_t.handle` a literal array index — `Cvar_Update` resolves a VM-side handle to a native cvar in O(1) with a bounds check, no hash lookup needed. The linked list (`cvar_vars`) enables ordered iteration for `cvarlist`, config writes, and `SetCheatState`; the hash table enables O(1) name lookup for every other operation. Both structures are maintained simultaneously.

- **Flag-bitmask protection hierarchy.** ROM → INIT → LATCH → CHEAT are checked in priority order inside `Cvar_Set2`. The `force` parameter bypasses this entirely, allowing the engine to override any cvar (e.g., for map-change latching). This is simpler than a callback/observer model and eliminates the need for per-cvar setter logic.

- **Global OR-accumulator `cvar_modifiedFlags`.** Rather than push-notifying subsystems on every cvar change, the engine polls a single int each frame. Server and client check specific bits (USERINFO, SERVERINFO, SYSTEMINFO) to know whether to rebroadcast info strings. This is polling over pub/sub — lower complexity, slightly higher latency (one frame), acceptable for configuration data.

- **LATCH deferred-apply pattern.** LATCH cvars (`com_soundmegs`, video resolution, etc.) that require a subsystem restart store the pending value in `latchedString` and apply it on the next `Cvar_Get` call for that name. This means `Cvar_Get` is not purely a registration function — it's also the deferred-apply trigger.

## Data Flow Through This File

```
Console input / exec script
        │
        ▼
Cvar_Command → Cvar_Set2 ──────────────────────────────────┐
                    │                                       │
C code: Cvar_Get ──►│◄── VM syscall: Cvar_Register         │
                    │    (handle = pool index stored in     │
                    ▼     vmCvar_t)                         │
        cvar_indexes[n]                                     │
          .string / .value / .integer                       │
          .flags (USERINFO, SERVERINFO, ...)                │
          .modified / .modificationCount                    │
          .latchedString (pending value)                    │
                    │                                       │
        ┌───────────┼───────────────────┐                  │
        ▼           ▼                   ▼                  │
Cvar_Update   Cvar_InfoString    Cvar_WriteVariables        │
(sync to       (→ userinfo /      (→ q3config.cfg)          │
 vmCvar_t)      serverinfo        via FS_Printf)            │
                configstrings                               │
                → network)                                  │
```

Key state transitions: `modified` is set on every value change and cleared (externally) by subsystems that poll it; `modificationCount` is a monotonically increasing generation counter used by `Cvar_Update` to avoid redundant VM copies.

## Learning Notes

- **The `vmCvar_t` bridge** is an instructive example of how Quake 3's sandbox VM architecture forces explicit data marshaling at every boundary. Game, cgame, and UI VMs cannot hold raw `cvar_t *` pointers (the VM address space is disjoint); instead they hold an integer handle that the engine resolves. This pattern predates modern language-level sandboxing but solves the same problem.

- **String-valued cvars with redundant numeric caches** (`value`, `integer` fields) reflect 1990s engine pragmatics: parsing `atof`/`atoi` on every read was considered too slow, so the parsed forms are cached at write time. Modern engines typically store a typed variant or parse lazily.

- **No per-cvar callbacks.** Unreal Engine 1 (contemporary) used delegates on config vars; Q3 relies entirely on `cvar_modifiedFlags` polling and `modificationCount` generation counters. Simpler, but means detection latency is always one frame.

- **`Cvar_InfoString` returns a static buffer** — this is a classic C reentrant-safety hazard that would be a design smell today. Callers must consume the result before making any other `Cvar_InfoString` call.

- **The `CVAR_USER_CREATED` promotion pattern** in `Cvar_Get` demonstrates how the engine handles boot ordering: a user can `set` a cvar in `autoexec.cfg` before the C module that owns it has initialized. When the module later calls `Cvar_Get`, the user value is preserved but the C default becomes the `resetString`, enabling `/cvar_restart` to return to a sane state.

## Potential Issues

- **Pool exhaustion is fatal (`ERR_FATAL`).** With `MAX_CVARS = 1024` and no reclamation (deleted cvars zero their slot but the index is not recycled), mods that register many dynamic cvars risk hitting this ceiling. `Cvar_Restart_f` does clear user-created cvars but cannot compact the pool.
- **`Cvar_InfoString` static buffer** is unsafe if called from multiple threads — relevant if a future SMP renderer thread ever queries serverinfo directly.
- **Hash quality is weak** for short names (polynomial over lowercase chars, 256 buckets). With ~200 registered cvars at runtime, collision chains will exist but are short enough not to matter in practice.
