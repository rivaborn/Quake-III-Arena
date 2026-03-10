# code/q3_ui/ui_main.c

## File Purpose
This is the Q3 UI module's entry point for the QVM virtual machine. It implements `vmMain`, the sole gateway through which the engine dispatches commands into the UI module, and manages the registration and updating of all UI-related cvars.

## Core Responsibilities
- Expose `vmMain` as the single engine-facing entry point for all UI commands
- Route engine UI commands (init, shutdown, input events, refresh, menu activation) to the appropriate handler functions
- Declare all UI-side `vmCvar_t` globals that mirror engine cvars
- Define a `cvarTable_t` table mapping cvar structs to their name, default, and flags
- Implement `UI_RegisterCvars` and `UI_UpdateCvars` to batch-register and sync all cvars

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `cvarTable_t` | struct | Associates a `vmCvar_t*` with its cvar name, default string, and flags for bulk registration |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `ui_ffa_fraglimit` … `ui_cdkeychecked` | `vmCvar_t` | global | Mirror structs for every UI-exposed engine cvar (game rules, server browser, SP progress, bookmarked servers, CD key, etc.) |
| `cvarTable` | `static cvarTable_t[]` | static | Table used to drive bulk cvar registration/update loops |
| `cvarTableSize` | `static int` | static | Computed count of entries in `cvarTable` |

## Key Functions / Methods

### vmMain
- **Signature:** `int vmMain( int command, int arg0–arg11 )`
- **Purpose:** QVM entry point — the engine calls this for every UI subsystem event.
- **Inputs:** `command` selects the operation; `arg0`–`arg11` are command-specific parameters.
- **Outputs/Return:** Varies by command: `UI_API_VERSION` for `UI_GETAPIVERSION`, `qtrue` for `UI_HASUNIQUECDKEY`, `0` for void commands, `-1` for unknown commands.
- **Side effects:** Delegates to `UI_Init`, `UI_Shutdown`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Refresh`, `UI_IsFullscreen`, `UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen`.
- **Calls:** `UI_Init`, `UI_Shutdown`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Refresh`, `UI_IsFullscreen`, `UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen`
- **Notes:** Must be the **first** function compiled into the `.qvm` file (per comment). The `UI_HASUNIQUECDKEY` case always returns `qtrue`; mod authors are advised to return `qfalse`.

### UI_RegisterCvars
- **Signature:** `void UI_RegisterCvars( void )`
- **Purpose:** Iterates `cvarTable` and calls `trap_Cvar_Register` for each entry to register all UI cvars with the engine.
- **Inputs:** None (reads `cvarTable` / `cvarTableSize`).
- **Outputs/Return:** `void`
- **Side effects:** Registers cvars with the engine; populates all `vmCvar_t` globals.
- **Calls:** `trap_Cvar_Register`

### UI_UpdateCvars
- **Signature:** `void UI_UpdateCvars( void )`
- **Purpose:** Iterates `cvarTable` and calls `trap_Cvar_Update` for each entry to refresh local `vmCvar_t` values from the engine.
- **Inputs:** None.
- **Outputs/Return:** `void`
- **Side effects:** Updates all `vmCvar_t` globals to current engine values.
- **Calls:** `trap_Cvar_Update`

## Control Flow Notes
- **Init:** `vmMain(UI_INIT)` → `UI_Init()` → calls `UI_RegisterCvars` (inferred from `ui_atoms.c`).
- **Per-frame:** `vmMain(UI_REFRESH, time)` → `UI_Refresh(time)` → calls `UI_UpdateCvars` to keep cvar mirrors in sync.
- **Shutdown:** `vmMain(UI_SHUTDOWN)` → `UI_Shutdown()`.
- This file has no frame loop of its own; all periodic work flows through engine-driven `vmMain` calls.

## External Dependencies
- `ui_local.h` — aggregates `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`, all menu/subsystem declarations, and all `trap_*` syscall prototypes.
- `UI_Init`, `UI_Shutdown`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Refresh`, `UI_IsFullscreen`, `UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen` — defined in `ui_atoms.c` / other `q3_ui` files.
- `trap_Cvar_Register`, `trap_Cvar_Update` — defined in `ui_syscalls.c`; bridge to engine via QVM syscall ABI.
- `UI_API_VERSION` — defined as `4` in `ui_local.h` (overrides the value from `ui_public.h`).
