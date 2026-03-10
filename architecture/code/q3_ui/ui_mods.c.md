# code/q3_ui/ui_mods.c

## File Purpose
Implements the Mods menu screen for Quake III Arena's legacy UI (`q3_ui`), allowing the player to browse installed game modifications and switch to one by setting `fs_game` and triggering a video restart.

## Core Responsibilities
- Enumerate available game mods via `trap_FS_GetFileList("$modlist", ...)`
- Populate a scrollable list UI widget with mod names and their directory names
- Handle "Go" action: write the selected mod's directory to `fs_game` cvar and execute `vid_restart`
- Handle "Back" action: pop the menu without making changes
- Pre-cache all menu artwork shaders on demand
- Register itself as a pushable menu via `UI_ModsMenu()`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `mods_t` | struct | Entire state for the mods menu: menu framework, UI widgets, and flat string buffers for mod names and game directories |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_mods` | `mods_t` | static (file) | Single persistent instance of the mods menu state; re-initialized on each `UI_Mods_MenuInit` call |

## Key Functions / Methods

### UI_Mods_MenuEvent
- **Signature:** `static void UI_Mods_MenuEvent(void *ptr, int event)`
- **Purpose:** Unified callback for all interactive widgets in the mods menu.
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is handled).
- **Outputs/Return:** void
- **Side effects:** On `ID_GO`: sets `fs_game` cvar, appends `vid_restart;` to the command buffer, pops the menu. On `ID_BACK`: pops the menu.
- **Calls:** `trap_Cvar_Set`, `trap_Cmd_ExecuteText`, `UI_PopMenu`
- **Notes:** Ignores all events other than `QM_ACTIVATED`. `vid_restart` triggers a full renderer/filesystem restart to load the new mod.

### UI_Mods_ParseInfos
- **Signature:** `static void UI_Mods_ParseInfos(char *modDir, char *modDesc)`
- **Purpose:** Appends one mod entry (directory name + description) into the pre-allocated flat string buffers and the list widget.
- **Inputs:** `modDir` — mod filesystem directory name (max 16 chars); `modDesc` — human-readable mod description (max 48 chars).
- **Outputs/Return:** void
- **Side effects:** Advances `s_mods.descriptionPtr` and `s_mods.fs_gamePtr` write cursors; increments `s_mods.list.numitems`.
- **Calls:** `Q_strncpyz`, `strlen`
- **Notes:** No bounds check against `MAX_MODS` here; the cap is enforced post-loop in `UI_Mods_LoadMods`.

### UI_Mods_LoadMods
- **Signature:** `static void UI_Mods_LoadMods(void)`
- **Purpose:** Queries the engine for all installed mods via the `$modlist` virtual path, seeds the list with the hardcoded "Quake III Arena" / `""` base entry, then parses each returned directory+description pair.
- **Inputs:** None (operates on `s_mods` global state)
- **Outputs/Return:** void
- **Side effects:** Populates `s_mods.list`, `s_mods.descriptionList`, `s_mods.fs_gameList`; prints count via `trap_Print`; clamps `numitems` to `MAX_MODS`.
- **Calls:** `trap_FS_GetFileList`, `UI_Mods_ParseInfos`, `trap_Print`, `strlen`
- **Notes:** The `dirlist` buffer returned by `trap_FS_GetFileList("$modlist", ...)` is a packed sequence of `dirName\0description\0` pairs; the pointer arithmetic in the loop walks this format.

### UI_Mods_MenuInit
- **Signature:** `static void UI_Mods_MenuInit(void)`
- **Purpose:** Zeroes and configures all widgets (banner, frame bitmaps, scroll list, back/go buttons) with hardcoded 640×480 layout coordinates, then loads mod data.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies `s_mods`; calls `UI_ModsMenu_Cache` and `UI_Mods_LoadMods`; registers all items with `Menu_AddItem`.
- **Calls:** `UI_ModsMenu_Cache`, `memset`, `UI_Mods_LoadMods`, `Menu_AddItem`
- **Notes:** `UI_ModsMenu_Cache` is called before `memset`, so cached shader handles are not stored in `s_mods` (they go through the renderer's shader table directly).

### UI_ModsMenu_Cache
- **Signature:** `void UI_ModsMenu_Cache(void)`
- **Purpose:** Pre-registers all six menu art shaders so they are resident before drawing begins.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Issues `trap_R_RegisterShaderNoMip` for each art asset.
- **Calls:** `trap_R_RegisterShaderNoMip` ×6

### UI_ModsMenu
- **Signature:** `void UI_ModsMenu(void)`
- **Purpose:** Public entry point; initializes and pushes the mods menu onto the UI stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `UI_Mods_MenuInit`, then `UI_PushMenu`.
- **Calls:** `UI_Mods_MenuInit`, `UI_PushMenu`

## Control Flow Notes
Called from the main menu or options flow via `UI_ModsMenu()`. On entry, `UI_Mods_MenuInit` fires once per open. The menu remains active until the user selects "Back" (silent pop) or "Go" (sets cvar + `vid_restart` + pop). There is no per-frame update logic; all interaction is callback-driven through `UI_Mods_MenuEvent`.

## External Dependencies
- **`ui_local.h`** — menu framework types (`menuframework_s`, `menulist_s`, `menubitmap_s`, `menutext_s`, `menucommon_s`), all `trap_*` syscall declarations, `UI_PushMenu`/`UI_PopMenu`, `Menu_AddItem`, `Q_strncpyz`, `color_white`
- **Defined elsewhere:** `trap_Cvar_Set`, `trap_Cmd_ExecuteText`, `trap_FS_GetFileList`, `trap_R_RegisterShaderNoMip`, `trap_Print`, `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — all resolved through the VM syscall layer at runtime.
