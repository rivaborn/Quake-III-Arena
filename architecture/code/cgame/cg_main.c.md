# code/cgame/cg_main.c

## File Purpose
This is the primary entry point and initialization module for the cgame (client-side game) VM module in Quake III Arena. It owns all global cgame state, registers cvars, and orchestrates the full asset precache pipeline during level load.

## Core Responsibilities
- Expose `vmMain()` as the sole entry point from the engine into the cgame VM
- Declare and own all global cgame state (`cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items`)
- Register and update all cgame `vmCvar_t` variables via a data-driven table
- Drive the level initialization sequence: sounds → graphics → clients → HUD
- Provide utility functions: `CG_Printf`, `CG_Error`, `CG_Argv`, `CG_ConfigString`
- Implement stub `Com_Error`/`Com_Printf` linkage for shared `q_shared.c`/`bg_*.c` code
- (MISSIONPACK) Load and initialize the script-driven HUD menu system via `displayContextDef_t`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `cvarTable_t` | struct | Associates a `vmCvar_t*`, its cvar name, default value, and flags for bulk registration |
| `displayContextDef_t` | struct (extern, ui_shared.h) | Function pointer table + assets used by the new HUD system (MISSIONPACK only) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cg` | `cg_t` | global | All transient per-frame client game state |
| `cgs` | `cgs_t` | global | All persistent client game static state (survives tournament restarts) |
| `cg_entities` | `centity_t[MAX_GENTITIES]` | global | Per-entity interpolation and rendering state |
| `cg_weapons` | `weaponInfo_t[MAX_WEAPONS]` | global | Weapon media handles and properties |
| `cg_items` | `itemInfo_t[MAX_ITEMS]` | global | Item model/icon handles |
| `cg_railTrailTime` … `cg_trueLightning` | `vmCvar_t` (many) | global | All cgame console variables |
| `cgDC` | `displayContextDef_t` | global (MISSIONPACK) | Display context for the new HUD/menu system |
| `forceModelModificationCount` | `int` | file-static (global) | Tracks `cg_forceModel` modification count to detect changes |
| `cvarTable` | `cvarTable_t[]` | static | Data-driven cvar registration table |
| `cvarTableSize` | `int` | static | Count of entries in `cvarTable` |

## Key Functions / Methods

### vmMain
- **Signature:** `int vmMain(int command, int arg0…arg11)`
- **Purpose:** Sole engine-to-cgame dispatch function; must be the first compiled symbol in the .q3vm.
- **Inputs:** `command` selects the operation (CG_INIT, CG_SHUTDOWN, CG_DRAW_ACTIVE_FRAME, etc.); `arg0`–`arg11` are operation-specific parameters.
- **Outputs/Return:** 0 on success, -1 on unknown command, or forwarded return values from `CG_CrosshairPlayer` / `CG_LastAttacker` / `CG_ConsoleCommand`.
- **Side effects:** Dispatches to all major subsystems; in MISSIONPACK builds, updates `cgDC.cursorx/y` on mouse events.
- **Calls:** `CG_Init`, `CG_Shutdown`, `CG_ConsoleCommand`, `CG_DrawActiveFrame`, `CG_CrosshairPlayer`, `CG_LastAttacker`, `CG_KeyEvent`, `CG_MouseEvent`, `CG_EventHandling`, `CG_Error`.
- **Notes:** This function is the ABI boundary. All cgame–engine communication flows through here.

### CG_Init
- **Signature:** `void CG_Init(int serverMessageNum, int serverCommandSequence, int clientNum)`
- **Purpose:** Full initialization after every level change or subsystem restart; performs all asset precaching.
- **Inputs:** Initial snapshot number, server command sequence, local client number.
- **Outputs/Return:** void.
- **Side effects:** Zeroes all global state; calls all Register* functions; loads collision map, sounds, graphics, clients, HUD (MISSIONPACK); starts background music; initializes local entities and mark polys.
- **Calls:** `memset` on all globals; `trap_R_RegisterShader`, `trap_CM_LoadMap`, `CG_RegisterCvars`, `CG_InitConsoleCommands`, `CG_RegisterSounds`, `CG_RegisterGraphics`, `CG_RegisterClients`, `CG_InitLocalEntities`, `CG_InitMarkPolys`, `CG_SetConfigValues`, `CG_StartMusic`, `CG_ShaderStateChanged`, `trap_S_ClearLoopingSounds`; (MISSIONPACK) `CG_AssetCache`, `CG_LoadHudMenu`, `String_Init`, `CG_InitTeamChat`.
- **Notes:** Sets `cg.loading = qtrue` before client registration to force immediate load rather than deferral.

### CG_RegisterCvars
- **Signature:** `void CG_RegisterCvars(void)`
- **Purpose:** Bulk-registers all cgame cvars from `cvarTable`, initializes `cgs.localServer`, and registers model/headmodel cvars.
- **Side effects:** Calls `trap_Cvar_Register` for each entry; reads `sv_running` to set `cgs.localServer`; snapshots `forceModelModificationCount`.
- **Calls:** `trap_Cvar_Register`, `trap_Cvar_VariableStringBuffer`.

### CG_UpdateCvars
- **Signature:** `void CG_UpdateCvars(void)`
- **Purpose:** Per-frame cvar refresh; detects changes to `cg_drawTeamOverlay` and `cg_forceModel`.
- **Side effects:** Updates all `vmCvar_t` via `trap_Cvar_Update`; toggles `teamoverlay` cvar; triggers `CG_ForceModelChange` when force-model changes. Contains a known "E3 HACK" that forces teamoverlay on.
- **Calls:** `trap_Cvar_Update`, `trap_Cvar_Set`, `CG_ForceModelChange`.

### CG_RegisterSounds / CG_RegisterGraphics / CG_RegisterClients
- **Notes:** Three large static functions called sequentially from `CG_Init`. Each registers its respective media type into `cgs.media`. `CG_RegisterGraphics` also loads the world map via `trap_R_LoadWorldMap` and registers all inline BSP sub-models. `CG_RegisterClients` calls `CG_NewClientInfo` for all connected players and builds the spectator scroll list.

### CG_LoadHudMenu (MISSIONPACK)
- **Signature:** `void CG_LoadHudMenu(void)`
- **Purpose:** Populates `cgDC` function pointer table and calls `Init_Display` + `CG_LoadMenus` to load the script-driven HUD.
- **Side effects:** Wires cgame rendering/sound/input functions into `cgDC`; resets and loads menu definitions from `cg_hudFiles`.
- **Calls:** `Init_Display`, `Menu_Reset`, `trap_Cvar_VariableStringBuffer`, `CG_LoadMenus`.

### CG_Printf / CG_Error
- **Notes:** Thin varargs wrappers over `trap_Print`/`trap_Error`. Also provide `Com_Printf`/`Com_Error` shims (compiled only when `CGAME_HARD_LINKED` is not defined) to satisfy link requirements of shared game code.

## Control Flow Notes
- **Init:** `vmMain(CG_INIT)` → `CG_Init` is the starting point after level load. It is synchronous and may take seconds (asset loading).
- **Frame:** `vmMain(CG_DRAW_ACTIVE_FRAME)` → `CG_DrawActiveFrame` (defined in `cg_view.c`). `CG_UpdateCvars` is called during the frame to pick up console variable changes.
- **Shutdown:** `vmMain(CG_SHUTDOWN)` → `CG_Shutdown` (currently a no-op stub).
- Input events (`CG_KEY_EVENT`, `CG_MOUSE_EVENT`, `CG_EVENT_HANDLING`) are forwarded directly from `vmMain`.

## External Dependencies
- `cg_local.h` — pulls in `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`, and all `trap_*` declarations
- `ui/ui_shared.h` (MISSIONPACK) — `displayContextDef_t`, menu system types, `Init_Display`, `Menu_*`, `PC_*` parse helpers
- **Defined elsewhere:** `CG_DrawActiveFrame`, `CG_ConsoleCommand`, `CG_NewClientInfo`, `CG_RegisterItemVisuals`, `CG_ParseServerinfo`, `CG_SetConfigValues`, `bg_itemlist`, `bg_numItems`, all `trap_*` syscall stubs
