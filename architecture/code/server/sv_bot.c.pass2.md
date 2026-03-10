# code/server/sv_bot.c — Enhanced Analysis

## Architectural Role

`sv_bot.c` is the **server-side integration adapter** between the Quake III authoritative server and the detached botlib AI navigation/decision library. It implements `botlib_import_t` (engine callbacks) and manages the versioned `botlib_export_t` vtable obtained at runtime, enabling bots to participate in normal server simulation while keeping the AI pipeline modular and optionally disabled. Bot clients are full citizens of the `svs.clients[]` array, subject to standard snapshot/command processing — they're indistinguishable from humans to snapshot building and net code.

## Key Cross-References

### Incoming (who depends on this)
- **`code/server/sv_main.c`** calls `SV_BotFrame(time)` each server tick to drive per-bot AI ticks into `gvm`
- **`code/server/sv_init.c`** calls `SV_BotInitCvars()` and `SV_BotInitBotLib()` during map load (part of `SV_SpawnServer` → `SV_InitGame` chain)
- **`code/game/g_bot.c` / `code/game/ai_*.c`** (game VM) invoke botlib functions via `trap_BotLib*` syscall range (opcodes 200–599), which index into `botlib_export` obtained here
- **Renderer** (via cgame) calls `BotDrawDebugPolygons` callback to visualize AAS geometry when `bot_debug` cvar is set

### Outgoing (what this file depends on)
- **`code/botlib/be_interface.c`**: calls `GetBotLibAPI(botlib_import_t, version)` to load botlib and obtain `botlib_export_t`; implements all `botlib_import_t` callbacks exposed to botlib
- **`code/qcommon/cm_*.c`**: collision subsystem (`SV_Trace`, `SV_ClipToEntity`, `SV_PointContents`, `CM_EntityString`, `CM_InlineModel`, `CM_ModelBounds`)
- **`code/qcommon/vm.c`**: `VM_Call(gvm, BOTAI_START_FRAME, time)` drives bot AI each frame
- **`code/server/sv_*.c`**: client state, snapshot building, reliable command queues, entity data
- **`code/qcommon/cmd.c`, `cvar.c`, `files.c`**: command dispatch, cvar registry, filesystem I/O

## Design Patterns & Rationale

**Dual-vtable plugin architecture:**  
Core insight is that botlib is **dlopen-ed or separately compiled** and must never be statically linked. Instead:
1. Engine populates `botlib_import_t` with all callbacks it provides
2. Engine calls `GetBotLibAPI(botlib_import_t, ...)` to load botlib and get back `botlib_export_t`  
3. Bidirectional calls flow through these two versioned vtables

Why? Botlib must be **portable offline** (`code/bspc` reuses it), so it can't depend on engine internals. This is a classic late-90s DLL plugin pattern — no modern reflection, just explicit function pointers.

**Mechanical adapter wrappers:**  
Functions like `BotImport_Trace` convert between engine internals (`trace_t`) and botlib's expected format (`bsp_trace_t`). Same for `BotImport_PointContents`, `BotImport_EntityTrace`. These hide schema differences without business logic.

**Lazy cvar binding in callbacks:**  
`BotDrawDebugPolygons` does `if (!bot_debug) bot_debug = Cvar_Get("bot_debug", ...)`. This is idiomatic Q3A — defers cvar lookup to first use, avoiding init-order bugs. Safe because cvar system is idempotent (multiple `Cvar_Get` calls for the same name return the same pointer).

**Master enable flag gating:**  
The global `bot_enable` flag acts as a circuit breaker. If `0`, `SV_BotFrame` returns immediately, and no botlib callbacks fire. Allows runtime bot disable without unloading botlib.

## Data Flow Through This File

1. **Init** → `SV_BotInitCvars()` registers ~25 bot cvars (feature gates, debug flags, tuning); `SV_BotInitBotLib()` allocs debug polygon pool, calls `GetBotLibAPI`, asserts success
2. **Per-frame** → `SV_BotFrame(time)` if enabled → `VM_Call(gvm, BOTAI_START_FRAME, time)` → game VM calls `trap_BotLib*` syscalls → index into `botlib_export` → botlib queries engine via `botlib_import_t` callbacks (trace, PVS, entity state)
3. **Debug** → `BotDrawDebugPolygons(drawPoly)` iterates live `debugpolygons[]` entries, dispatches to renderer callback; optionally runs botlib's `Test` visualizer
4. **Client mgmt** → `SV_BotAllocateClient` finds free `svs.clients[]` slot, sets `CS_ACTIVE`/`NA_BOT`; bot then flows through normal snapshot/command pipelines; `SV_BotFreeClient` reclaims slot

## Learning Notes

**Late-90s best practices on display:**  
Botlib's boundary is a **masterclass in plugin isolation without modern DI** — vtable pointers passed at load time, no global state shared across the boundary, version negotiation built in. Contrast with modern engines (Unreal, Unity) that use scriptable reflection or dependency containers.

**Debug infrastructure as first-class citizen:**  
The `debugpolygons` pool and `BotDrawDebugPolygons` callback aren't bolted on; they're architected into the core. Idiomatic to Q3A's emphasis on shipping internal tools (debugging, profiling) alongside product code.

**Movement model baked in:**  
Botlib's reachability calculations assume Q3A's specific movement physics (gravity, step height, jump arc). Modern engines abstract pathfinding from movement — botlib is tightly coupled here, reflecting the era's constraints.

**Entirely optional subsystem:**  
Unlike Unreal/Unity where AI is core, Q3A's botlib is optional and fully gated by `bot_enable`. Reflects the game's focus on competitive multiplayer (human-vs-human); bots were post-launch feature.

## Potential Issues

**Unsafe pointer writes in callbacks:** `BotImport_Trace` writes to `*bsptrace` without null-check. If botlib passes garbage, engine state corrupts. Modern code would assert or validate.

**Debug code assumes single-player slot 0:** `BotDrawDebugPolygons` reads `svs.clients[0].gentity->r.currentOrigin` — breaks in dedicated servers or if bot occupies slot 0. Leaky assumption, not a crash, but fragile.

**No SMP safety on debug polygon pool:** Concurrent renderer front-end and botlib modifying `debugpolygons[]` could race on multi-threaded systems. Original engine was single-threaded, so not an issue then.

**Silent failure on botlib load:** If `GetBotLibAPI` returns NULL, code asserts; no graceful degradation. Production build would crash rather than disable bots with a warning.
