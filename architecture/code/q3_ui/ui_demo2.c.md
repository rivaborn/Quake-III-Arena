# code/q3_ui/ui_demo2.c

## File Purpose
Implements the Demos menu for Quake III Arena's legacy UI module (`q3_ui`). It scans the `demos/` directory for demo files matching the current protocol version, populates a scrollable list, and allows the player to play a selected demo or navigate back.

## Core Responsibilities
- Initialize and lay out all widgets for the Demos menu screen
- Enumerate demo files via `trap_FS_GetFileList` filtered by protocol-versioned extension (e.g., `dm_68`)
- Strip file extensions and uppercase demo names for display
- Handle user interaction: play selected demo, navigate list left/right, go back
- Preload/cache all menu artwork shaders via `Demos_Cache`
- Guard against the empty-list degenerate case by disabling the "Go" button

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `demos_t` | struct | Entire state for the Demos menu: menu framework, all widget instances, demo name buffer, and pointer array |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_demos` | `demos_t` | static (file) | Single persistent instance of the Demos menu state |

## Key Functions / Methods

### Demos_MenuEvent
- **Signature:** `static void Demos_MenuEvent( void *ptr, int event )`
- **Purpose:** Callback dispatched by the menu system for widget activation events
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** void
- **Side effects:** On `ID_GO`, clears the menu stack and issues a `demo <name>` console command via `trap_Cmd_ExecuteText`. On `ID_BACK`, pops the menu. On `ID_LEFT`/`ID_RIGHT`, scrolls the list.
- **Calls:** `UI_ForceMenuOff`, `trap_Cmd_ExecuteText`, `va`, `UI_PopMenu`, `ScrollList_Key`
- **Notes:** Silently ignores all non-`QM_ACTIVATED` events.

### UI_DemosMenu_Key
- **Signature:** `static sfxHandle_t UI_DemosMenu_Key( int key )`
- **Purpose:** Per-frame key handler registered on the menu framework; delegates to default menu key processing
- **Inputs:** `key` — raw key code
- **Outputs/Return:** `sfxHandle_t` sound to play (from `Menu_DefaultKey`)
- **Side effects:** None beyond what `Menu_DefaultKey` produces
- **Calls:** `Menu_ItemAtCursor` (result unused), `Menu_DefaultKey`
- **Notes:** `item` is retrieved but never used — dead code left in place.

### Demos_MenuInit
- **Signature:** `static void Demos_MenuInit( void )`
- **Purpose:** Zeroes and fully configures `s_demos`, enumerates demo files, builds the item name pointer array, and registers all widgets with the menu framework
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes to global `s_demos`; calls `trap_FS_GetFileList` (I/O); calls `trap_Cvar_VariableValue("protocol")` to build the file extension; mutates `s_demos.names` buffer (strips `.dm3` suffix, uppercases names)
- **Calls:** `memset`, `Demos_Cache`, `Com_sprintf`, `trap_FS_GetFileList`, `trap_Cvar_VariableValue`, `strcpy`, `strlen`, `Q_stricmp`, `Q_strupr`, `Menu_AddItem`
- **Notes:** Clamps `numitems` to `MAX_DEMOS` (128). If no demos are found, inserts a placeholder string and hides/disables the Go button. Extension stripping checks for `.dm3` only (4-char suffix), not the protocol-versioned extension used for file listing — potential mismatch for non-`.dm3` builds.

### Demos_Cache
- **Signature:** `void Demos_Cache( void )`
- **Purpose:** Pre-registers all menu artwork shaders so they are resident before the menu draws
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for 9 art assets
- **Calls:** `trap_R_RegisterShaderNoMip` (×9)
- **Notes:** Called during `Demos_MenuInit` and also callable externally for precaching during level load.

### UI_DemosMenu
- **Signature:** `void UI_DemosMenu( void )`
- **Purpose:** Public entry point — initializes and pushes the Demos menu onto the UI stack
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies `s_demos` (via `Demos_MenuInit`); pushes onto the global menu stack (via `UI_PushMenu`)
- **Calls:** `Demos_MenuInit`, `UI_PushMenu`

## Control Flow Notes
`UI_DemosMenu` is called from the main menu or in-game menu when the player selects "Demos." It pushes the menu onto the `uis.stack`. Each frame, the UI system calls the framework's draw and key callbacks. Widget events flow through `Demos_MenuEvent`. No per-frame update logic exists; the demo list is static after init.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `trap_*` syscall wrappers (`ui_syscalls.c`)
  - `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` (`ui_atoms.c`)
  - `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (`ui_qmenu.c`)
  - `ScrollList_Key` (`ui_qmenu.c`)
  - `Q_stricmp`, `Q_strupr`, `Com_sprintf`, `va` (`q_shared.c`)
  - `color_white` (global color constant, `ui_qmenu.c`)
