# code/q3_ui/ui_menu.c

## File Purpose
Implements the Quake III Arena main menu screen, including menu item layout, 3D banner model rendering, error message display, and navigation to all top-level game sections.

## Core Responsibilities
- Initializes and configures the `mainmenu_t` menu item list at startup
- Handles menu item activation events, routing to sub-menus (SP, MP, Setup, Demos, Cinematics, Mods, Exit)
- Renders the 3D rotating banner model in the menu background using the renderer API
- Displays `com_errorMessage` as an overlay when the engine reports an error
- Conditionally shows the "TEAM ARENA" option when the `missionpack` mod directory exists
- Performs CD key validation on startup and redirects to the CD key menu if invalid
- Draws copyright/demo watermark strings at the bottom of the screen

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `mainmenu_t` | struct | Aggregates the `menuframework_s` and all `menutext_s` items for the main menu, plus the banner model handle |
| `errorMessage_t` | struct | Holds a separate `menuframework_s` and a 4096-byte error message buffer for engine error display |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_main` | `mainmenu_t` | static (file) | Persistent main menu state; zeroed on each `UI_MainMenu()` call |
| `s_errorMessage` | `errorMessage_t` | static (file) | Persistent error overlay state; zeroed on each `UI_MainMenu()` call |

## Key Functions / Methods

### MainMenu_ExitAction
- **Signature:** `static void MainMenu_ExitAction( qboolean result )`
- **Purpose:** Confirmation callback for the "EXIT GAME?" dialog; transitions to the credits screen on confirmation.
- **Inputs:** `result` — `qtrue` if user confirmed exit
- **Outputs/Return:** void
- **Side effects:** Calls `UI_PopMenu()` and `UI_CreditMenu()`
- **Calls:** `UI_PopMenu`, `UI_CreditMenu`
- **Notes:** Early-returns silently if `result` is false (user cancelled).

---

### Main_MenuEvent
- **Signature:** `void Main_MenuEvent( void *ptr, int event )`
- **Purpose:** Callback dispatched by the menu framework when any main menu item is activated; routes to the appropriate sub-menu or action.
- **Inputs:** `ptr` — pointer to the `menucommon_s` that fired; `event` — event type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** void
- **Side effects:** May call various `UI_*Menu()` functions, set cvars (`fs_game`), or queue console commands (`vid_restart`)
- **Calls:** `UI_SPLevelMenu`, `UI_ArenaServersMenu`, `UI_SetupMenu`, `UI_DemosMenu`, `UI_CinematicsMenu`, `UI_ModsMenu`, `UI_ConfirmMenu`, `trap_Cvar_Set`, `trap_Cmd_ExecuteText`
- **Notes:** `ID_TEAMARENA` switches `fs_game` to `"missionpack"` and triggers `vid_restart` — this is a hard engine restart path.

---

### MainMenu_Cache
- **Signature:** `void MainMenu_Cache( void )`
- **Purpose:** Preloads the 3D banner model into the renderer.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes `s_main.bannerModel` via `trap_R_RegisterModel`
- **Calls:** `trap_R_RegisterModel`

---

### ErrorMessage_Key
- **Signature:** `sfxHandle_t ErrorMessage_Key( int key )`
- **Purpose:** Key handler for the error overlay menu; any keypress clears `com_errorMessage` and returns to the main menu.
- **Inputs:** `key` — keycode (ignored)
- **Outputs/Return:** `menu_null_sound`
- **Side effects:** Clears `com_errorMessage` cvar; re-enters `UI_MainMenu()`
- **Calls:** `trap_Cvar_Set`, `UI_MainMenu`

---

### Main_MenuDraw
- **Signature:** `static void Main_MenuDraw( void )`
- **Purpose:** Per-frame draw function shared by both the main menu and error overlay; renders the 3D banner, menu items or error text, and copyright strings.
- **Inputs:** None (reads `s_main`, `s_errorMessage`, `uis`)
- **Outputs/Return:** void
- **Side effects:** Submits render commands via trap calls; time-driven sine animation on the banner yaw
- **Calls:** `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `trap_R_RenderScene`, `UI_DrawProportionalString_AutoWrapped`, `Menu_Draw`, `UI_DrawProportionalString`, `UI_DrawString`, `UI_AdjustFrom640`, `AxisClear`, `AnglesToAxis`, `VectorSet`, `VectorCopy`
- **Notes:** `RDF_NOWORLDMODEL` is set so the scene renders without a BSP world. The viewport is fixed at 640×120 virtual pixels. A commented-out sine wobble on FOV (`adjust`) was removed at Kenneth's request.

---

### UI_TeamArenaExists
- **Signature:** `static qboolean UI_TeamArenaExists( void )`
- **Purpose:** Queries the mod directory list to determine whether the `missionpack` mod is installed.
- **Inputs:** None
- **Outputs/Return:** `qtrue` if `"missionpack"` is found in the mod list
- **Side effects:** None
- **Calls:** `trap_FS_GetFileList`, `Q_stricmp`
- **Notes:** Iterates a flat packed string list from `$modlist`; each entry is `name\0description\0`.

---

### UI_MainMenu
- **Signature:** `void UI_MainMenu( void )`
- **Purpose:** Top-level entry point; validates the CD key, zeroes state, configures all menu items, conditionally adds Team Arena, and pushes the menu onto the UI stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Sets `sv_killserver` cvar; zeroes `s_main` and `s_errorMessage`; pushes menu via `UI_PushMenu`; sets key catcher to `KEYCATCH_UI`; resets `uis.menusp` to 0
- **Calls:** `trap_Cvar_Set`, `trap_GetCDKey`, `trap_VerifyCDKey`, `UI_CDKeyMenu`, `MainMenu_Cache`, `trap_Cvar_VariableStringBuffer`, `UI_TeamArenaExists`, `Menu_AddItem`, `trap_Key_SetCatcher`, `UI_PushMenu`
- **Notes:** If `com_errorMessage` is non-empty, a stripped-down error menu is pushed instead of the full main menu. CD key check is skipped in demo mode or if already checked (`ui_cdkeychecked`).

## Control Flow Notes
`UI_MainMenu` is called during UI initialization and after returning from sub-menus or error states. It resets the entire menu stack (`uis.menusp = 0`) rather than pushing on top of an existing stack. `Main_MenuDraw` is registered as the `menu.draw` callback and is invoked each UI refresh frame by the menu framework.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types)
- **Defined elsewhere:** `uis` (`uiStatic_t` global from `ui_atoms.c`), all `trap_*` syscall wrappers (`ui_syscalls.c`), `Menu_Draw`/`Menu_AddItem` (`ui_qmenu.c`), all `UI_*Menu()` navigation targets (their respective `.c` files), `color_red`/`menu_text_color`/`menu_null_sound` (`ui_qmenu.c`), `ui_cdkeychecked` (`ui_main.c`)
