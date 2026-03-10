# code/botlib/be_interface.c — Enhanced Analysis

## Architectural Role

`be_interface.c` implements the **module entry point and initialization backbone** for a self-contained bot AI library within Quake III's plugin architecture. It bridges the server engine's view of botlib (as a swappable DLL exposing `GetBotLibAPI`) and the internal subsystem hierarchy by populating a versioned `botlib_export_t` function-pointer table, orchestrating sequential initialization of AAS (spatial), EA (primitives), and AI (decision-making) layers, and maintaining global state that gates all subsystem access. The file mediates a clean dependency inversion: botlib imports engine services (`botlib_import_t`: file I/O, collision, debug rendering) rather than linking against the engine, enabling portability and runtime modularity.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/server/sv_bot.c`** — Calls `GetBotLibAPI(BOTLIB_API_VERSION, &botimport)` once at bot system init; stores the returned `botlib_export_t*` for the lifetime of the server instance
- **`code/game/ai_*.c` (game VM)** — Does not link to `be_interface.c` directly; instead, the game VM calls botlib functions exclusively through `trap_BotLib*` syscalls (opcodes 200–599), which the server routes to functions exported by this module's export table
- **`code/server/sv_game.c`** — Hosts the game VM and provides `botimport` callback functions (e.g., `trap_Trace`, `FS_ReadFile`, `PVS` queries) that `be_interface.c` passes to botlib

### Outgoing (what this file depends on)

- **All `code/botlib/be_aas_*.c` modules** — `AAS_Setup`, `AAS_LoadMap`, `AAS_StartFrame`, `AAS_UpdateEntity`, `AAS_Shutdown` are the core spatial query subsystem
- **`code/botlib/be_ea.c`** — Elementary actions (`EA_Setup`, `EA_Shutdown`, `EA_GetInput`) that synthesize per-bot `bot_input_t` from AI decisions
- **`code/botlib/be_ai_*.c` modules** — Higher-level AI subsystems (`BotSetupWeaponAI`, `BotSetupGoalAI`, `BotSetupChatAI`, `BotSetupMoveAI` and their shutdown counterparts)
- **`code/botlib/l_*.c` modules** — Utility stack: `l_libvar` (config), `l_log` (file logging), `l_precomp` (script preprocessor), `l_memory` (custom alloc)
- **`botimport` (engine callbacks)** — `Print`, `DebugLineCreate`, `Trace`, `FS_ReadFile`, `GetEntityState`, `LinkEntity`, and PVS queries; all filled in by `GetBotLibAPI` at load time

## Design Patterns & Rationale

**Versioned Plugin ABI**: The `GetBotLibAPI(apiVersion)` entry point is a classic binary compatibility pattern that validates API version before populating the export table. This avoids silent incompatibility mismatches and allows multiple binary versions to coexist, though it does not support version negotiation (e.g., "I support v1 through v3")—version must match exactly. Pragmatic for Quake 3's era (no reflection, minimal overhead).

**Nested Namespace Organization**: The export table is hierarchical (`aas_export_t`, `ea_export_t`, `ai_export_t` nested in `botlib_export_t`), which mirrors the subsystem tree and avoids function name collisions—every exported function is scoped by its table. Modern engines use namespacing or fully qualified names; this is the struct-based equivalent.

**Hierarchical Sequential Initialization**: AAS → EA → WeaponAI → GoalAI → ChatAI → MoveAI ensures that spatial and primitive layers are ready before higher-level AI initializes. Shutdown is strictly reverse order. This is a classical hierarchical resource lifecycle pattern; initialization fails-fast on first error (no rollback or partial recovery).

**Import/Export Inversion**: Rather than linking against the engine, botlib declares what it *needs* (`botlib_import_t`) and *provides* (`botlib_export_t`), then the server populates both at load time. This avoids circular linking and makes botlib portable—it can be embedded in tools (`bspc`) or other projects. The engine's dependency on botlib is explicit (one call to `GetBotLibAPI`) rather than scattered across function calls.

**Redundant Setup Flags**: Both `botlibsetup` (file-static) and `botlibglobals.botlibsetup` are maintained identically. This suggests incomplete refactoring—likely the global was added later but the static flag was never removed. Harmless but a code smell.

**Libvar Mini-System**: Configuration is via `LibVarSet`/`LibVarGet` rather than the engine's cvars, suggesting botlib was designed as a portable, standalone module before tight engine integration. The libvar system is custom and isolated within botlib.

## Data Flow Through This File

**Load-Time Initialization:**
- Engine loads botlib DLL and calls `GetBotLibAPI(BOTLIB_API_VERSION, &server_import_table)`
- `GetBotLibAPI` validates version, stores `botimport` (engine callbacks), zeroes `be_botlib_export`, calls `Init_AAS_Export`, `Init_EA_Export`, `Init_AI_Export` to populate function pointers
- Returns `&be_botlib_export` to server; server stores it

**Setup Time (once per server instance):**
- Server calls `Export_BotLibSetup()` (via export table)
- Zeroes `botlibglobals`, opens `botlib.log`, reads `maxclients`/`maxentities` libvars
- Sequentially calls `AAS_Setup`, `EA_Setup`, `BotSetupWeaponAI`, `BotSetupGoalAI`, `BotSetupChatAI`, `BotSetupMoveAI`
- Each may fail; first error returns immediately (no partial cleanup, relies on shutdown to handle)
- Sets `botlibsetup=qtrue` on success

**Map Load (once per level):**
- Server calls `Export_BotLibLoadMap(mapname)` 
- Calls `AAS_LoadMap(mapname)` — reads `.aas` binary file, validates checksums, initializes spatial indexes
- Calls `BotInitLevelItems`, `BotSetBrushModelTypes` to precompute level-specific data

**Per-Frame:**
- Server calls `Export_BotLibStartFrame(time)` → `AAS_StartFrame(time)` (updates frame delta)
- Per entity: `Export_BotLibUpdateEntity(entnum, state)` → `AAS_UpdateEntity` (syncs BSP entity/mover positions into AAS)
- Game VM calls `trap_BotLib*` syscalls, routed to functions in `botlib_export_t.ai` sub-table

**Shutdown (once at server end):**
- Server calls `Export_BotLibShutdown()`
- Reverse-order shutdown of all subsystems: ChatAI, MoveAI, GoalAI, WeaponAI, Weights, Characters, AAS, EA
- Frees libvars, precompiler defines, closes log
- Sets `botlibsetup=qfalse`

## Learning Notes

**Idiomatic patterns of Quake 3 era:**

1. **Binary ABI versioning without negotiation**: `GetBotLibAPI` requires exact version match rather than a "compatible versions" range. Simple to implement, strict enforcement, no hidden incompatibilities.

2. **Dependency inversion via callback tables**: Rather than botlib calling engine functions directly, the engine provides a `botlib_import_t` callback table. This is a manual form of dependency injection, common before frameworks automated it. Makes botlib testable and portable.

3. **Static-only internal exports**: All functions in `Init_*_Export` are static-scoped; only `GetBotLibAPI` and the exported functions themselves are visible outside the DLL. This is deliberate—external callers should only use the export table, not hunt for internal functions.

4. **Custom config system (libvar)**: Rather than relying on the engine's cvar system, botlib has its own `LibVarSet`/`Get`. This isolation suggests botlib was originally a standalone library (indeed, tools like `bspc` use it without the engine).

5. **Fail-fast initialization**: If any subsystem init fails, the whole setup aborts. No recovery, no partial initialization. Modern engines might attempt recovery or degraded modes; this assumes: "if botlib can't init, the server can't run."

6. **DEBUG-exclusive testing hook**: `BotExportTest` is exposed in the public export table but its body is `#ifdef DEBUG`; release builds return a no-op. Pragmatic in-engine debugging without release-build overhead.

7. **No abstract base classes or inheritance**: All subsystems are procedural; the export table is a flat struct of function pointers, not a vtable. This is simple, explicit, and fits a C codebase without OOP.

## Potential Issues

1. **Partial initialization on failure**: If `BotSetupGoalAI()` fails, `AAS_Setup` and `EA_Setup` are left in an initialized state. The code does not roll back earlier subsystems; cleanup is deferred to shutdown. If shutdown is called, it cleans everything, but the intermediate state is fragile.

2. **Global guard pattern**: Every caller must check `BotLibSetup()` or the `botlibsetup` flag. This is necessary but makes isolated testing of subsystems difficult and introduces a global dependency that would be a code smell in modern architectures (though appropriate for Quake 3).

3. **Exact version matching**: `apiVersion != BOTLIB_API_VERSION` returns `NULL`. There is no version compatibility negotiation. If the engine and botlib DLL are built against different API headers, the mismatch is detected at runtime but cannot be gracefully degraded.

4. **Redundant flag maintenance**: Both the static `botlibsetup` and `botlibglobals.botlibsetup` are kept in sync. This is a refactoring artifact and suggests the code was not fully cleaned up during evolution.
