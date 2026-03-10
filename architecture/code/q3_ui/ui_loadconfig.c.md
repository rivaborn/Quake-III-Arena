# code/q3_ui/ui_loadconfig.c

## File Purpose
Implements the "Load Config" UI menu for Quake III Arena, allowing the player to browse and execute `.cfg` configuration files found in the game's file system.

## Core Responsibilities
- Initializes and lays out the Load Config menu's UI widgets (banner, frame art, scrollable file list, navigation arrows, back/go buttons)
- Enumerates all `.cfg` files via the filesystem trap and populates a scrollable list
- Strips `.cfg` extensions and uppercases filenames for display
- Handles user interactions: executing the selected config, navigating the list, or dismissing the menu
- Pre-caches all menu art shaders via `UI_LoadConfig_Cache`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `configs_t` | struct | All state for the Load Config menu: menu framework, UI widgets, filename buffer, and pointer array for the scroll list |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_configs` | `configs_t` | static (file) | Single instance holding all menu state; zeroed and re-initialized on each menu open |

## Key Functions / Methods

### LoadConfig_MenuEvent
- **Signature:** `static void LoadConfig_MenuEvent( void *ptr, int event )`
- **Purpose:** Callback for all interactive widgets; dispatches on widget ID to execute a config, pop the menu, or scroll the list.
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — callback event type
- **Outputs/Return:** void
- **Side effects:** On `ID_GO`, appends an `exec <filename>` command to the command buffer and pops the menu. On `ID_BACK`, pops the menu. On `ID_LEFT`/`ID_RIGHT`, calls `ScrollList_Key` to scroll the list.
- **Calls:** `trap_Cmd_ExecuteText`, `va`, `UI_PopMenu`, `ScrollList_Key`
- **Notes:** Early-returns for any event that is not `QM_ACTIVATED`.

### LoadConfig_MenuInit
- **Signature:** `static void LoadConfig_MenuInit( void )`
- **Purpose:** Zeros and fully initializes `s_configs`, builds all widget descriptors, scans for `.cfg` files, and registers items with the menu framework.
- **Inputs:** None (reads from filesystem)
- **Outputs/Return:** void
- **Side effects:** Calls `UI_LoadConfig_Cache` to register shaders; mutates `s_configs.names` in-place (null-terminates after stripping extensions, uppercases names); calls `Menu_AddItem` nine times.
- **Calls:** `UI_LoadConfig_Cache`, `memset`, `trap_FS_GetFileList`, `strcpy`, `strlen`, `Q_stricmp`, `Q_strupr`, `Menu_AddItem`
- **Notes:** Caps list at `MAX_CONFIGS` (128). If no files are found, inserts a "No Files Found." placeholder and disables/hides the Go button. Extension stripping operates directly on the shared `names` buffer — the pointer arithmetic (`configname += len + 1`) walks past the null terminator written by the strip, relying on `trap_FS_GetFileList` packing names with a trailing `\0` each.

### UI_LoadConfig_Cache
- **Signature:** `void UI_LoadConfig_Cache( void )`
- **Purpose:** Pre-registers all menu art shaders with the renderer so they are resident before drawing.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Issues nine `trap_R_RegisterShaderNoMip` calls.
- **Calls:** `trap_R_RegisterShaderNoMip` (×9)
- **Notes:** Called both from `LoadConfig_MenuInit` and potentially by the global cache-warming path.

### UI_LoadConfigMenu
- **Signature:** `void UI_LoadConfigMenu( void )`
- **Purpose:** Public entry point; initializes the menu and pushes it onto the menu stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `LoadConfig_MenuInit` then `UI_PushMenu`.
- **Calls:** `LoadConfig_MenuInit`, `UI_PushMenu`

## Control Flow Notes
This file is entirely UI/menu layer code. It has no frame or update loop of its own. `UI_LoadConfigMenu` is called on demand (e.g., from the Options menu). Once pushed, the generic menu system drives drawing and input; widget callbacks (`LoadConfig_MenuEvent`) handle all transitions. The file is not involved in init/render/shutdown of the engine itself.

## External Dependencies
- **`ui_local.h`** — pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, all menu type definitions, and all `trap_*` syscall declarations
- **Defined elsewhere:** `trap_Cmd_ExecuteText`, `trap_FS_GetFileList`, `trap_R_RegisterShaderNoMip`, `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `ScrollList_Key`, `va`, `Q_stricmp`, `Q_strupr`, `color_white` — all resolved from other UI/qcommon translation units at link time
