# code/q3_ui/ui_playermodel.c

## File Purpose
Implements the Player Model selection menu in the Quake III Arena q3_ui module. It scans the filesystem for available player model/skin icons, presents them in a paginated 4×4 grid, and persists the selected model/skin to CVars on exit.

## Core Responsibilities
- Build a list of available player models by scanning `models/players/*/icon_*.tga` files
- Render a paginated 4×4 grid of model portrait bitmaps with navigation arrows
- Track the currently selected model/skin, displaying its name and skin name as text
- Render a live 3D player preview using `UI_DrawPlayer` (owner-draw callback)
- Save the selected model to `model`, `headmodel`, `team_model`, and `team_headmodel` CVars
- Handle keyboard navigation (arrow keys, page turning) and mouse clicks on portrait buttons
- Guard 3D player rendering behind a `LOW_MEMORY` (5 MB) threshold

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `playermodel_t` | struct | All state for the player model menu: menu framework, portrait bitmaps, pagination state, selected model index, modelskin string, playerInfo |
| `playerInfo_t` | struct (extern, from ui_local.h) | Full player rendering info: model handles, lerp frames, weapon, view/move angles |
| `menubitmap_s` | struct (extern) | Generic bitmap menu item with focus/shader/dimensions |
| `menutext_s` | struct (extern) | Text menu item with string pointer, style, color |
| `menuframework_s` | struct (extern) | Container for all menu items, cursor, key callback |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_playermodel` | `playermodel_t` | static (file) | Sole instance of all player model menu state |
| `playermodel_artlist` | `char*[]` | static (file) | NULL-terminated list of art shader paths to precache |

## Key Functions / Methods

### PlayerModel_BuildList
- **Signature:** `static void PlayerModel_BuildList(void)`
- **Purpose:** Populates `s_playermodel.modelnames[]` by iterating `models/players/` subdirectories and collecting `icon_*.tga` skin files.
- **Inputs:** None (reads filesystem via `trap_FS_GetFileList`)
- **Outputs/Return:** void; fills `s_playermodel.modelnames`, sets `nummodels`, `numpages`, resets `modelpage`
- **Side effects:** Calls `trap_S_RegisterSound` for announce sounds when `com_buildscript` is set; modifies global `s_playermodel`
- **Calls:** `trap_FS_GetFileList`, `trap_Cvar_VariableValue`, `COM_StripExtension`, `Q_stricmpn`, `Com_sprintf`, `trap_S_RegisterSound`, `va`
- **Notes:** Caps at `MAX_PLAYERMODELS` (256). Directory trailing slash is stripped in-place. Off-by-one bug exists: `nummodels` is incremented before being used as the size index in `Com_sprintf`.

### PlayerModel_SetMenuItems
- **Signature:** `static void PlayerModel_SetMenuItems(void)`
- **Purpose:** Reads current `model` and `name` CVars, finds the matching entry in `modelnames[]`, and initializes `selectedmodel`, `modelpage`, `modelname.string`, and `skinname.string`.
- **Inputs:** None (reads CVars)
- **Outputs/Return:** void
- **Side effects:** Modifies `s_playermodel` selection/page fields and static string buffers
- **Calls:** `trap_Cvar_VariableStringBuffer`, `Q_CleanStr`, `Q_strncpyz`, `strstr`, `strcat`, `Q_stricmp`, `Q_strupr`, `strlen`

### PlayerModel_UpdateGrid
- **Signature:** `static void PlayerModel_UpdateGrid(void)`
- **Purpose:** Refreshes `pics[]` and `picbuttons[]` flags/names for the current page; highlights the selected item; enables/disables left/right arrow buttons.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies `s_playermodel.pics[].generic.name`, `.flags`, `.shader`, and `picbuttons[].generic.flags`
- **Calls:** None (pure struct manipulation)

### PlayerModel_PicEvent
- **Signature:** `static void PlayerModel_PicEvent(void* ptr, int event)`
- **Purpose:** Callback fired when a portrait button is activated; updates selection state, parses model/skin name from the icon path, and triggers a model reload if memory allows.
- **Inputs:** `ptr` — pointer to the activated `menucommon_s`; `event` — QM_ notification type
- **Outputs/Return:** void
- **Side effects:** Writes to `s_playermodel.modelskin`, `modelname.string`, `skinname.string`, `selectedmodel`; conditionally calls `PlayerModel_UpdateModel`
- **Calls:** `Q_strncpyz`, `strcat`, `strstr`, `strlen`, `Q_strupr`, `trap_MemoryRemaining`, `PlayerModel_UpdateModel`

### PlayerModel_UpdateModel
- **Signature:** `static void PlayerModel_UpdateModel(void)`
- **Purpose:** Resets and re-initializes `playerinfo` for the current `modelskin`, posing the model at idle/stand with a machinegun.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** `memset` clears `playerinfo`; calls `UI_PlayerInfo_SetModel` and `UI_PlayerInfo_SetInfo`

### PlayerModel_DrawPlayer
- **Signature:** `static void PlayerModel_DrawPlayer(void *self)`
- **Purpose:** Owner-draw callback for the player preview bitmap; guards against low memory by showing a warning string instead.
- **Inputs:** `self` — `menubitmap_s*`
- **Calls:** `trap_MemoryRemaining`, `UI_DrawProportionalString`, `UI_DrawPlayer`

### PlayerModel_SaveChanges
- **Signature:** `static void PlayerModel_SaveChanges(void)`
- **Purpose:** Commits `modelskin` to all four model CVars.
- **Calls:** `trap_Cvar_Set` ×4

### PlayerModel_MenuKey
- **Signature:** `static sfxHandle_t PlayerModel_MenuKey(int key)`
- **Purpose:** Custom key handler enabling left/right arrow navigation across portrait grid with automatic page turns on grid edges.
- **Calls:** `Menu_ItemAtCursor`, `Menu_SetCursor`, `PlayerModel_UpdateGrid`, `PlayerModel_SaveChanges`, `Menu_DefaultKey`

### PlayerModel_MenuInit
- **Signature:** `static void PlayerModel_MenuInit(void)`
- **Purpose:** Zeroes state, calls cache/build/set helpers, lays out all menu items with hardcoded 640×480 coordinates, and adds them to the framework.
- **Calls:** `memset`, `PlayerModel_Cache`, `Menu_AddItem` ×many, `PlayerModel_SetMenuItems`, `PlayerModel_UpdateGrid`, `PlayerModel_UpdateModel`
- **Notes:** `PlayerModel_BuildList()` call is commented out here; it is called inside `PlayerModel_Cache` instead.

### PlayerModel_Cache
- **Signature:** `void PlayerModel_Cache(void)`
- **Purpose:** Precaches all UI art shaders and all discovered model portrait shaders; also drives `PlayerModel_BuildList`.
- **Calls:** `trap_R_RegisterShaderNoMip`, `PlayerModel_BuildList`

### UI_PlayerModelMenu
- **Signature:** `void UI_PlayerModelMenu(void)`
- **Purpose:** Public entry point; inits the menu, pushes it onto the UI stack, and positions cursor on the currently selected model.
- **Calls:** `PlayerModel_MenuInit`, `UI_PushMenu`, `Menu_SetCursorToItem`

## Control Flow Notes
- **Init:** `UI_PlayerModelMenu` → `PlayerModel_MenuInit` → `PlayerModel_Cache` (loads art + builds model list) → `PlayerModel_SetMenuItems` (finds current CVar selection) → `PlayerModel_UpdateGrid` + `PlayerModel_UpdateModel`
- **Frame:** `PlayerModel_DrawPlayer` is invoked each frame via the owner-draw hook on the player bitmap item during `Menu_Draw`
- **Input:** `PlayerModel_MenuKey` intercepts arrow keys for grid navigation; `PlayerModel_PicEvent` handles portrait activation
- **Shutdown:** Back button / Escape calls `PlayerModel_SaveChanges` then `UI_PopMenu`

## External Dependencies
- `ui_local.h` — includes `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types, trap syscall declarations
- **Defined elsewhere:** `UI_DrawPlayer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo` (ui_players.c); `Menu_*` functions (ui_qmenu.c); all `trap_*` syscalls (ui_syscalls.c); `uis` global (ui_atoms.c); `menu_move_sound`, `menu_buzz_sound` (ui_qmenu.c)
