# code/q3_ui/ui_main.c — Enhanced Analysis

## Architectural Role

This file is the **exclusive entry point** for the UI subsystem. It implements the QVM contract via `vmMain`, serving as a synchronous command dispatcher that bridges the engine's client layer (`code/client/cl_ui.c`) to the entire UI module. Beyond initialization, it manages the critical cross-subsystem cvar synchronization layer, acting as the coordinator between engine-authoritative cvar state and UI-local mirrors—essential because the UI must read values written by the engine and game VM (e.g., `g_spScores*`, `cg_brassTime`) while exposing its own settings back to the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Engine/Client** (`code/client/cl_ui.c`): Calls `vmMain` with all UI commands (`UI_INIT`, `UI_REFRESH`, `UI_KEY_EVENT`, `UI_MOUSE_EVENT`, `UI_DRAW_CONNECT_SCREEN`, `UI_SET_ACTIVE_MENU`)—the only way UI code executes
- **Game VM** (`code/game/g_main.c`, indirectly): May trigger menu state changes via `UI_SET_ACTIVE_MENU` during level load or gameplay state transitions
- **cgame VM** (`code/cgame/cg_main.c`, indirectly): May invoke UI_SET_ACTIVE_MENU for postgame scoreboard display

### Outgoing (what this file depends on)
- **Other q3_ui modules** (`ui_atoms.c`, `ui_qmenu.c`, etc.): `UI_Init`, `UI_Shutdown`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Refresh`, `UI_IsFullscreen`, `UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen` all defined elsewhere
- **Engine syscalls** (`code/qcommon/vm.c` dispatch): `trap_Cvar_Register`, `trap_Cvar_Update` bridge to engine cvar system
- **Shared headers**: `ui_local.h` aggregates `ui_public.h` (which defines `UI_INIT`, `UI_REFRESH` command constants), `q_shared.h`, keycodes

## Design Patterns & Rationale

**Command Dispatcher**: `vmMain`'s switch statement is the canonical QVM entry pattern—single, deterministic function ensuring all engine→UI transitions are logged/instrumented. Engine always knows the exact UI API version (`UI_API_VERSION`) and can validate compatibility.

**Data-Driven Cvar Registration**: The `cvarTable` array decouples cvar metadata (name, default, flags) from registration logic. This reduces boilerplate and centralizes the "UI configuration schema"—critical for mod authorship (e.g., changing `UI_HASUNIQUECDKEY`'s return value) and for coordinating with `ui/menudef.h` constants.

**Deferred Sync**: `UI_UpdateCvars` called per-frame (from `UI_Refresh`) ensures UI code always reads the freshest engine state without polling. This is essential because engine cvars (e.g., `cg_drawCrosshair`, `g_spScores1`) can be modified by the server, console, or cgame VM—the UI must stay in sync.

**Mixed Ownership**: The cvarTable includes cvars from three sources:
- **UI-owned** (`ui_ffa_fraglimit`, `ui_browserMaster`): UI preserves user preferences
- **Game-owned** (`g_arenasFile`, `g_spScores*`, `g_spSkill`): Server/game writes; UI reads for SP progression display
- **cgame-owned** (`cg_brassTime`, `cg_drawCrosshair`, `cg_marks`): cgame state the UI must expose in settings menus

This coordination is not explicit in the code—it relies on naming convention and shared header `ui/menudef.h`.

## Data Flow Through This File

```
Engine time-domain flow:
  1. vmMain(UI_INIT)
     → UI_Init()
       → UI_RegisterCvars()  [populate all vmCvar_t globals, set defaults]
       → [rest of UI init]

  2. Per frame: vmMain(UI_REFRESH, time)
     → UI_Refresh(time)
       → UI_UpdateCvars()  [sync all vmCvar_t from engine]
       → [render, input processing]

  3. Input: vmMain(UI_KEY_EVENT, key, down)
     → UI_KeyEvent(key, down)

  4. vmMain(UI_SHUTDOWN)
     → UI_Shutdown()

Cvar data flow:
  Engine holds "authoritative" cvar state
  → UI calls trap_Cvar_Register(cvarName, defaultString, flags)  [at init]
  → Engine initializes vmCvar_t struct, stores value internally
  → UI calls trap_Cvar_Update(vmCvar_t)  [each frame]
  → vmCvar_t.value, vmCvar_t.string updated in-place by engine
  → UI code reads ui_ffa_fraglimit.integer, ui_spScores1.string, etc.
  → Some UI writes (e.g., button clicks) invoke trap_Cvar_Set, modifying engine state
```

## Learning Notes

**QVM Entry Point Convention**: This file exemplifies the required structure for QVM modules (cgame, game, ui all follow this pattern). The `vmMain` signature with 12 arg slots is wasteful but ensures ABI stability across module reloads. Modern engines use variadic arguments or tagged unions; Q3A's design predates that.

**Cvar as Cross-VM Communication**: The UI uses cvars as a **remote procedure call abstraction**—setting a cvar from the UI effectively sends a message to the engine or game VM, which poll cvars they care about. This is more flexible than direct syscalls for loosely-coupled state (e.g., UI doesn't know if `ui_spSkill` is read by game, cgame, or both).

**API Versioning**: `UI_API_VERSION` (value 4 per `ui_local.h`) allows the engine to reject incompatible UI VMs before calling further functions. The game VM, cgame, and renderer all use the same pattern—a safety mechanism from the era of downloadable mods.

**CVAR_ROM Semantics**: Read-only cvars (`ui_cdkeychecked`, `ui_spSelection`) prevent the menu from modifying certain state; the engine or game enforces invariants by returning ROM flags.

**Table-Size Computation**: The use of `sizeof(cvarTable) / sizeof(cvarTable[0])` is idiomatic C for array iteration. No bounds checking in the registration loop—assumes `cvarTable` is correctly terminated (no sentinel), relying on compile-time verification.

## Potential Issues

1. **Unknown Command Returns -1**: `vmMain`'s default case returns `-1` for unrecognized commands. The engine may not handle this gracefully (silent failure vs. assert); mods extending the UI API need to ensure engine and UI are in sync.

2. **UI_HASUNIQUECDKEY Always qtrue**: The comment acknowledges this: mod authors must edit the source and recompile. No clean extensibility mechanism for CD-key validation (reflects Q3A's post-release mod scene where recompiling was expected).

3. **No Cvar Validation**: `UI_RegisterCvars` blindly registers all cvars without checking for duplicates or missing `trap_*` implementations. If a cvar registration fails (bad name, out of memory), the error propagates silently into the registered vmCvar_t (engine sets it to defaults); UI code won't know.

4. **Cvar Sync Dependency**: `UI_UpdateCvars` must be called **every frame** to maintain consistency. If `UI_Refresh` is ever skipped (e.g., minimized window), the UI stales relative to engine state. No change-tracking or dirty-flag mechanism to optimize the loop.

5. **Implicit Cvar Sharing**: The cvarTable mixes UI, game, and cgame cvars with no documentation. Renaming or removing a cvar breaks mod compatibility silently—no compile-time or runtime check.
