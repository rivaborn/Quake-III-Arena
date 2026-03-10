# code/botlib/be_interface.h — Enhanced Analysis

## Architectural Role

This file declares the **boundary interface** between the engine and botlib. It defines the singleton global state (`botlib_globals_t`) that botlib reads each frame from the server, and exposes the `botlib_import_t` vtable through which botlib makes all reverse calls into the engine. Fundamentally, it establishes botlib as a **plugin module** with unidirectional dependency: botlib depends on engine services, but engine never links to botlib symbols directly—only through the `botlib_export_t` function-pointer table obtained at runtime.

## Key Cross-References

### Incoming (who depends on this file)

- **Server (`code/server/sv_bot.c`, `sv_main.c`)**: Drives per-frame botlib ticks; manages `botlib_import_t` injection at startup and calls `BotAI_*` functions via the exported interface
- **Game VM (`code/game/g_bot.c`)**: Calls `trap_BotLib*` syscalls (opcodes 200–599), which the server routes to botlib; consults `botlibglobals.botlibsetup` to gate bot operations
- **botlib implementation (`code/botlib/be_*.c`)**: All AAS, pathfinding, AI modules read `botlibglobals` to access current `time`, entity/client limits, and debug state

### Outgoing (what this file depends on)

- **Platform layer**: Calls `Sys_MilliSeconds()` (implemented in `unix/unix_main.c`, `win32/win_main.c`, or `macosx/macosx_sys.m`)
- **botlib_import_t type** (declared in `code/botlib/botlib.h`): Engine callback vtable passed in at `BotLibSetup`; includes file I/O, memory, BSP traces, PVS queries, entity state
- **q_shared.h**: `vec3_t`, `qboolean` types for DEBUG fields

## Design Patterns & Rationale

**Plugin Module Pattern**: botlib never imports engine code; instead, the engine passes a callback table (`botimport`) that botlib invokes. This decoupling allows:
- botlib to be compiled/tested independently or even in offline tools (bspc)
- Engine to load multiple AI implementations by swapping the DLL at runtime
- No circular dependencies between engine and AI layers

**Global Frame State**: Rather than passing `time` and entity limits as function parameters, botlib reads them from a singleton each frame. This is idiomatic for 1999-era engines and avoids deep call-stack threading. The FIXME comment acknowledges this is not modern OOP design but was acceptable for performance and API stability.

**DEBUG Conditional**: Debug fields are compiled in only if `#define DEBUG` is uncommented. This allows shipping builds to omit goalarea tracking and AI step-through without recompiling botlib, while developers can rebuild with debug support for troubleshooting.

**RANDOMIZE Always On**: Unlike DEBUG, `RANDOMIZE` is unconditionally defined, meaning bot behavior is always subject to stochastic variance. There's no compile-time switch to disable it; the variance is baked in (likely in `be_ai_gen.c` or `be_ai_weap.c` where fuzzy logic scoring happens).

## Data Flow Through This File

**Initialization**:
1. Engine calls `GetBotLibAPI()` (DLL export), passing engine's `botlib_import_t` vtable
2. botlib's `BotLibSetup()` (via `botlib_export_t`) stores `botimport` globally and sets `botlibglobals.botlibsetup = qtrue`

**Per-Frame**:
1. Server updates `botlibglobals.time` to the current server frame timestamp (e.g., via `Com_GetRealTime()`)
2. Server calls `BotAI_StartFrame()` (per-bot or global, depending on version)
3. All AAS/pathfinding/AI logic in botlib reads `botlibglobals.time` and uses it as a consistent clock
4. botlib invokes `botimport` callbacks (e.g., `botimport->Trace()`, `botimport->GetEntityState()`) to query the world state

**Shutdown**:
1. Server calls `BotLibShutdown()`
2. botlib cleans up; `botlibglobals.botlibsetup = qfalse`

## Learning Notes

**Engine-Module Integration Pattern**: This file exemplifies how late-90s game engines decoupled subsystems. Rather than monolithic interdependencies, the engine defines a contract (import/export vtables) and lets modules implement them independently. This pattern appears elsewhere (renderer as swappable DLL, UI as QVM).

**Frame-Driven Simulation**: The presence of a global `time` field reflects the synchronous, frame-based architecture: all AI decisions within a frame see the same simulated time. There's no async threading or time-dilation; botlib is ballpark-synchronous with server ticks.

**Platform Abstraction via Stubs**: `Sys_MilliSeconds()` is declared here but never called by botlib itself—it's likely an artifact of botlib's shared code lineage (used in tools like `bspc`, or intended for timestamp logging). The real time reference is the engine's `botlibglobals.time`.

**Finite World Bounds**: `maxentities` and `maxclients` constraints hint that botlib pre-allocates entity links/caches at init time rather than dynamically. This is typical for predictable, fixed-size embedded systems (console games of the era).

**Optional Debug Instrumentation**: The `#ifdef DEBUG` fields (`goalareanum`, `goalorigin`, `runai`) suggest botlib had built-in visualization/introspection for AI debugging—developers could inspect what goal a bot was pursuing or step through the FSM.

## Potential Issues

- **Global State Not Thread-Safe**: If the engine ever threads the server frame loop, concurrent access to `botlibglobals` and `botimport` will race. The design assumes single-threaded server.
- **Time Synchronization Bug Risk**: If server updates `botlibglobals.time` inconsistently (e.g., sometimes to server frame, sometimes to real time), botlib's pathfinding caches and movement prediction could diverge. No obvious defensive code here guards against that.
- **RANDOMIZE Cannot Be Disabled at Runtime**: The macro is unconditionally defined, so bots always have stochastic behavior. If gameplay requires deterministic bot behavior (replays, competitive), there's no knob to turn off randomness—a rebuild is needed.
