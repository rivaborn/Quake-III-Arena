# code/q3_ui/keycodes.h
## File Purpose
Defines the `keyNum_t` enumeration mapping all recognized input sources (keyboard, mouse, joystick, aux) to integer key codes for use by the input and UI systems. It serves as the shared vocabulary for key event dispatch throughout the Q3 UI module.

## Core Responsibilities
- Enumerate all virtual key codes for keyboard special keys, function keys, numpad keys, mouse buttons, scroll wheel, joystick buttons, and auxiliary inputs
- Anchor low-ASCII printable/control keys at their ASCII values (TAB=9, ENTER=13, ESC=27, SPACE=32)
- Provide `K_LAST_KEY` as a sentinel/bounds-check value (must remain < 256)
- Define `K_CHAR_FLAG` bitmask to multiplex character events over the same key-event path

## External Dependencies
- No includes.
- `keyNum_t` values are consumed by: `KeyEvent` (defined elsewhere in the client/input layer), menu/UI event handlers (defined elsewhere in `q3_ui/`).
- `K_CHAR_FLAG` (value `1024`) is used by the menu code to distinguish char vs. key events — the or'ing logic lives outside this file.

# code/q3_ui/ui_addbots.c
## File Purpose
Implements the in-game "Add Bots" menu for Quake III Arena, allowing players to add AI bots to a running server session. It builds a scrollable list of available bots with skill level and team selection controls.

## Core Responsibilities
- Initialize and display the Add Bots menu UI with all interactive widgets
- Retrieve and alphabetically sort available bot names from the game's bot info database
- Scroll a paginated list of up to 7 bot names at a time
- Handle bot selection highlighting and dispatch the `addbot` server command on confirmation
- Pre-cache all menu art assets for rendering
- Adapt team options based on the current game type (FFA vs. team modes)

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types, trap syscalls)
- **Defined elsewhere:**
  - `UI_GetBotInfoByNumber`, `UI_GetNumBots` — `ui_gameinfo.c`
  - `Menu_Draw`, `Menu_AddItem`, `Menu_AddItem` — `ui_qmenu.c`
  - `UI_PushMenu`, `UI_PopMenu`, `UI_DrawBannerString`, `UI_DrawNamedPic` — `ui_atoms.c`
  - `trap_*` syscall wrappers — `ui_syscalls.c`
  - `Info_ValueForKey`, `Q_strncpyz`, `Q_stricmp`, `Com_Clamp` — `q_shared.c`

# code/q3_ui/ui_atoms.c
## File Purpose
Core UI module for Quake III Arena's legacy menu system (`q3_ui`), providing the foundational drawing primitives, menu stack management, input dispatch, and per-frame refresh logic used by all menu screens.

## Core Responsibilities
- Maintain and manage the menu stack (`UI_PushMenu`, `UI_PopMenu`, `UI_ForceMenuOff`)
- Dispatch keyboard and mouse input events to the active menu
- Draw proportional (bitmap font) strings in multiple styles (normal, banner, shadow, pulse, inverse, wrapped)
- Draw fixed-width strings with Quake color code support
- Provide 640×480 virtual-coordinate primitives (`UI_FillRect`, `UI_DrawRect`, `UI_DrawHandlePic`, etc.)
- Initialize and refresh the UI system each frame (`UI_Init`, `UI_Refresh`)
- Route console commands to specific menu entry points (`UI_ConsoleCommand`)

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- `trap_*` syscall wrappers (defined in `ui_syscalls.c`) — all renderer, sound, key, cvar, and cmd operations
- `Menu_Cache`, `Menu_Draw`, `Menu_DefaultKey`, `Menu_SetCursor` — defined in `ui_qmenu.c`
- `g_color_table`, `Q_IsColorString`, `ColorIndex` — defined in `q_shared.c`
- All `UI_*Menu()` and `*_Cache()` functions — defined in their respective `ui_*.c` files

# code/q3_ui/ui_cdkey.c
## File Purpose
Implements the CD Key entry menu for Quake III Arena's legacy UI system. It allows the player to enter, validate, and submit a 16-character CD key, integrating with the engine's CD key storage and verification syscalls.

## Core Responsibilities
- Initialize and lay out the CD Key menu using the `menuframework_s` widget system
- Render a custom owner-draw field displaying the CD key input with real-time format feedback
- Pre-validate the CD key format client-side (length + allowed character set)
- Store a confirmed key via `trap_SetCDKey` on acceptance
- Pre-populate the field from the engine via `trap_GetCDKey`, clearing it if verification fails
- Cache menu artwork shaders for reuse
- Expose public entry points (`UI_CDKeyMenu`, `UI_CDKeyMenu_f`, `UI_CDKeyMenu_Cache`) consumed by the rest of the UI module

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`
- **Defined elsewhere:**
  - `trap_SetCDKey`, `trap_GetCDKey`, `trap_VerifyCDKey` — engine syscall wrappers (`ui_syscalls.c`)
  - `trap_R_RegisterShaderNoMip`, `trap_Key_GetOverstrikeMode`, `trap_Cvar_Set` — engine syscall wrappers
  - `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — `ui_atoms.c` / `ui_qmenu.c`
  - `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`, `UI_DrawProportionalString` — `ui_atoms.c`
  - `uis` (`uiStatic_t`) — global UI state, `ui_atoms.c`
  - `color_yellow`, `color_orange`, `color_white`, `color_red`, `listbar_color` — `ui_qmenu.c`
  - `BIGCHAR_WIDTH`, `BIGCHAR_HEIGHT` — defined in shared UI headers

# code/q3_ui/ui_cinematics.c
## File Purpose
Implements the Cinematics menu for the Quake III Arena UI, allowing players to replay pre-rendered RoQ cutscene videos (id logo, intro, tier completions, and ending). It builds and presents a scrollable text-button list that triggers `disconnect; cinematic <name>.RoQ` commands when activated.

## Core Responsibilities
- Define and initialize all menu items for the Cinematics screen (banner, frame art, text buttons, back button)
- Gray out tier cinematic entries that the player has not yet unlocked via `UI_CanShowTierVideo`
- Handle back-navigation by popping the menu stack
- On item activation, set the `nextmap` cvar and issue a disconnect + cinematic playback command
- Handle the demo version special case for the "END" cinematic
- Expose a console-command entry point (`UI_CinematicsMenu_f`) that also repositions the cursor to a specific item
- Precache menu art shaders via `UI_CinematicsMenu_Cache`

## External Dependencies
- **`ui_local.h`** — pulls in all menu framework types, trap syscalls, `uis` global, `UI_CanShowTierVideo`, `UI_PopMenu`, `UI_PushMenu`, `va`, `color_red`, `color_white`, `QMF_*`, `QM_ACTIVATED`, `MTYPE_*`
- **Defined elsewhere:** `UI_CanShowTierVideo` (`ui_gameinfo.c`), `UI_PopMenu`/`UI_PushMenu` (`ui_atoms.c`), all `trap_*` syscall wrappers (`ui_syscalls.c`), `Menu_AddItem`/`Menu_SetCursorToItem` (`ui_qmenu.c`), `uis` global state (`ui_atoms.c`)

# code/q3_ui/ui_confirm.c
## File Purpose
Implements a reusable modal confirmation dialog and message box for the Quake III Arena legacy UI (q3_ui). It presents a yes/no prompt or a multi-line informational message overlaid on the current screen, invoking a callback with the user's boolean result.

## Core Responsibilities
- Display a modal yes/no confirmation dialog with a question string
- Display a modal message box with multiple text lines and a single "OK" button
- Route keyboard input (`Y`/`N`, arrow keys, tab) to the appropriate menu items
- Pop the menu from the stack and invoke a caller-supplied callback with the result
- Cache the confirmation frame artwork via `trap_R_RegisterShaderNoMip`
- Support an optional custom draw callback for additional overlay rendering
- Determine fullscreen vs. overlay mode based on connection state

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`)
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `Menu_DefaultKey`, `Menu_Draw`, `UI_DrawNamedPic`, `UI_DrawProportionalString`, `UI_ProportionalStringWidth`, `trap_R_RegisterShaderNoMip`, `trap_GetClientState`, `color_red`, key constants (`K_TAB`, `K_LEFTARROW`, etc.)

# code/q3_ui/ui_connect.c
## File Purpose
Renders the connection/loading screen shown while the client connects to a server. Handles display of connection state transitions, active file download progress, and ESC-key disconnection.

## Core Responsibilities
- Draw the full-screen connection overlay (background, server name, map name, MOTD)
- Display per-state status text (challenging, connecting, awaiting gamestate)
- Show real-time download progress: file size, transfer rate, estimated time remaining
- Track the last connection state to reset loading text on regression
- Handle the ESC key during connection to issue a disconnect command

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- **Defined elsewhere:** `trap_GetClientState`, `trap_GetConfigString`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cmd_ExecuteText`, `Menu_Cache`, `UI_SetColor`, `UI_DrawHandlePic`, `UI_DrawProportionalString`, `UI_DrawProportionalString_AutoWrapped`, `UI_ProportionalStringWidth`, `UI_ProportionalSizeScale`, `Info_ValueForKey`, `Com_sprintf`, `va`, `uis` (global `uiStatic_t`)

# code/q3_ui/ui_controls2.c
## File Purpose
Implements the full Controls configuration menu for Quake III Arena's legacy UI (q3_ui). It manages keyboard binding assignment, mouse/joystick configuration cvars, a live player model preview, and tabbed section navigation (Move/Look/Shoot/Misc).

## Core Responsibilities
- Define and manage the complete keybinding table (`g_bindings[]`) for all player actions
- Read current key bindings from the engine and populate local store (`Controls_GetConfig`)
- Write modified bindings and cvars back to the engine (`Controls_SetConfig`)
- Handle the "waiting for key" input capture state for rebinding
- Animate a 3D player model preview in response to focused action items
- Organize controls into four tabbed sections with dynamic show/hide of menu items
- Support resetting all bindings and cvars to defaults via a confirmation dialog

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h`, `keycodes.h`, `tr_types.h`
- **Defined elsewhere:** `trap_Key_*`, `trap_Cvar_*`, `trap_R_RegisterModel/Shader`, `trap_Cmd_ExecuteText` (syscall stubs in `ui_syscalls.c`); `UI_PlayerInfo_SetModel/SetInfo`, `UI_DrawPlayer` (`ui_players.c`); `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (`ui_qmenu.c`); `UI_ConfirmMenu` (`ui_confirm.c`); `bg_itemlist` (`bg_misc.c`)

# code/q3_ui/ui_credits.c
## File Purpose
Implements the credits screen menu for Quake III Arena's legacy UI (`q3_ui`). It renders a static list of id Software team members and pushes itself onto the menu stack as a fullscreen menu that quits the game on any keypress.

## Core Responsibilities
- Define and register the credits menu structure with the UI menu system
- Draw all credit text (roles and names) using proportional string rendering
- Handle key input by triggering a game quit command
- Push the credits screen onto the active menu stack as fullscreen

## External Dependencies
- **Includes:** `ui_local.h` → transitively pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`
- **Defined elsewhere:**
  - `UI_DrawProportionalString`, `UI_DrawString` — `ui_atoms.c`
  - `UI_PushMenu` — `ui_atoms.c`
  - `trap_Cmd_ExecuteText` — `ui_syscalls.c`
  - `color_white`, `color_red` — `ui_qmenu.c`
  - `menuframework_s`, `K_CHAR_FLAG`, `PROP_HEIGHT`, `PROP_SMALL_SIZE_SCALE`, `SMALLCHAR_HEIGHT` — `ui_local.h` / `q_shared.h`

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

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `trap_*` syscall wrappers (`ui_syscalls.c`)
  - `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` (`ui_atoms.c`)
  - `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (`ui_qmenu.c`)
  - `ScrollList_Key` (`ui_qmenu.c`)
  - `Q_stricmp`, `Q_strupr`, `Com_sprintf`, `va` (`q_shared.c`)
  - `color_white` (global color constant, `ui_qmenu.c`)

# code/q3_ui/ui_display.c
## File Purpose
Implements the Display Options menu screen within Quake III Arena's legacy UI module (`q3_ui`). It presents two hardware-facing sliders (brightness and screen size) alongside navigation tabs to sibling option screens (Graphics, Sound, Network).

## Core Responsibilities
- Initialize and lay out all widgets for the Display Options menu
- Pre-cache art assets (frame bitmaps, back button) at load time
- Map slider values to `r_gamma` and `cg_viewsize` cvars on activation
- Navigate to sibling option menus (Graphics, Sound, Network) via `UI_PopMenu` + push
- Gray out the brightness slider when the GPU does not support gamma (`uis.glconfig.deviceSupportsGamma`)
- Expose `UI_DisplayOptionsMenu` and `UI_DisplayOptionsMenu_Cache` as the public API for this screen

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `uis` (`uiStatic_t`) — global UI state, provides `glconfig.deviceSupportsGamma`
  - `Menu_AddItem`, `Menu_SetCursorToItem`, `UI_PushMenu`, `UI_PopMenu` — menu framework (`ui_qmenu.c` / `ui_atoms.c`)
  - `UI_GraphicsOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu` — sibling screen entry points
  - `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip` — VM syscall wrappers (`ui_syscalls.c`)
  - `color_red`, `color_white`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT` — shared UI constants

# code/q3_ui/ui_gameinfo.c
## File Purpose
Manages loading, parsing, and querying arena and bot metadata for the Quake III Arena UI module. Also tracks and persists single-player game progression, award data, and tier video unlock state via cvars.

## Core Responsibilities
- Load and parse arena info from `.arena` files and `scripts/arenas.txt` into a pool allocator
- Load and parse bot info from `.bot` files and `scripts/bots.txt`
- Assign ordered indices to arenas, separating single-player, special, and FFA arenas
- Query arena/bot records by number, map name, or special tag
- Read and write single-player scores per skill level via `g_spScores1–5` cvars
- Track award totals and tier cinematic unlock state via cvars
- Provide cheat/debug commands to unlock all levels and medals

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h` — shared types, `vmCvar_t`, `qboolean`, info-string API
- `COM_Parse`, `COM_ParseExt` — defined in `qcommon`
- `Info_SetValueForKey`, `Info_ValueForKey` — defined in `q_shared.c`
- `trap_*` syscalls — defined in `ui_syscalls.c`, dispatched into the engine VM interface
- `UI_SPLevelMenu_ReInit` — defined in `ui_spLevel.c`
- `uis` (`uiStatic_t`) — global UI state defined in `ui_atoms.c`

# code/q3_ui/ui_ingame.c
## File Purpose
Implements the in-game pause menu for Quake III Arena, presenting a vertical list of text buttons that allow the player to access game management options (team, bots, setup, server info, restart, quit, resume, leave) while paused mid-session.

## Core Responsibilities
- Define and initialize all menu items (`ingamemenu_t`) for the in-game overlay menu
- Conditionally gray out menu items based on runtime cvars (e.g., `sv_running`, `bot_enable`, `g_gametype`)
- Dispatch UI navigation events to the appropriate sub-menu or game command via `InGame_Event`
- Pre-cache the frame background shader via `InGame_Cache`
- Reset menu stack to top-level and push the initialized menu via `UI_InGameMenu`

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types)
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `UI_ConfirmMenu`, `UI_CreditMenu`, `UI_TeamMainMenu`, `UI_SetupMenu`, `UI_ServerInfoMenu`, `UI_AddBotsMenu`, `UI_RemoveBotsMenu`, `UI_TeamOrdersMenu`, `Menu_AddItem`, `trap_*` syscall wrappers, `uis` global, `color_red`, `Info_ValueForKey`

# code/q3_ui/ui_loadconfig.c
## File Purpose
Implements the "Load Config" UI menu for Quake III Arena, allowing the player to browse and execute `.cfg` configuration files found in the game's file system.

## Core Responsibilities
- Initializes and lays out the Load Config menu's UI widgets (banner, frame art, scrollable file list, navigation arrows, back/go buttons)
- Enumerates all `.cfg` files via the filesystem trap and populates a scrollable list
- Strips `.cfg` extensions and uppercases filenames for display
- Handles user interactions: executing the selected config, navigating the list, or dismissing the menu
- Pre-caches all menu art shaders via `UI_LoadConfig_Cache`

## External Dependencies
- **`ui_local.h`** — pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, all menu type definitions, and all `trap_*` syscall declarations
- **Defined elsewhere:** `trap_Cmd_ExecuteText`, `trap_FS_GetFileList`, `trap_R_RegisterShaderNoMip`, `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `ScrollList_Key`, `va`, `Q_stricmp`, `Q_strupr`, `color_white` — all resolved from other UI/qcommon translation units at link time

# code/q3_ui/ui_local.h
## File Purpose
Central internal header for the legacy `q3_ui` UI module. It declares all shared types, constants, extern variables, and function prototypes used across the UI subsystem's many `.c` source files.

## Core Responsibilities
- Define the menu-item type system (`MTYPE_*`) and flag bitmask (`QMF_*`) constants
- Declare all menu widget structs (`menuframework_s`, `menucommon_s`, `menufield_s`, `menuslider_s`, `menulist_s`, etc.)
- Declare the top-level UI state singleton `uiStatic_t uis`
- Expose `vmCvar_t` extern declarations for all UI-owned cvars
- Declare the full set of `trap_*` syscall wrappers used by UI VM code
- Forward-declare all per-screen cache/init/draw entry points across every UI screen file
- Declare the `playerInfo_t` / `lerpFrame_t` types used for 3D player preview rendering

## External Dependencies
- `game/q_shared.h` — core types (`vec3_t`, `qboolean`, `vmCvar_t`, `sfxHandle_t`, etc.)
- `cgame/tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t`)
- `ui/ui_public.h` — `uiExport_t`, `uiImport_t`, `uiMenuCommand_t`, `uiClientState_t` (imported from new UI; `UI_API_VERSION` overridden to 4)
- `keycodes.h` — `keyNum_t` enum, `K_CHAR_FLAG`
- `game/bg_public.h` — `weapon_t`, `animation_t`, `animNumber_t`, `MAX_ANIMATIONS`, game-type enums
- **Defined elsewhere:** All `trap_*` bodies (`ui_syscalls.c`), all `Menu_*` / `Bitmap_*` / `ScrollList_*` bodies (`ui_qmenu.c`), all per-screen `*_Cache` / `UI_*Menu` functions in their respective `.c` files.

# code/q3_ui/ui_login.c
## File Purpose
Implements the in-game login menu screen for Quake III Arena's online rankings system (GRank). It presents a modal dialog with name and password fields, wiring up input to the rankings authentication syscall.

## Core Responsibilities
- Define and initialize all UI widgets for the login form (frame, labels, text fields, buttons)
- Handle `LOGIN` and `CANCEL` button events via `Login_MenuEvent`
- Submit credentials to the rankings backend via `trap_CL_UI_RankUserLogin`
- Preload/cache the frame shader asset via `Login_Cache`
- Push the menu onto the UI stack via `UI_LoginMenu`

## External Dependencies
- **`ui_local.h`** — pulls in all menu types, trap wrappers, and helper declarations
- `trap_CL_UI_RankUserLogin` — defined in `ui_syscalls.c`/engine; submits credentials to the rankings server (not declared in the bundled header, implying it is a raw syscall wrapper unique to the GRank module)
- `trap_R_RegisterShaderNoMip` — renderer syscall
- `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` — defined in `ui_atoms.c`
- `Menu_AddItem` — defined in `ui_qmenu.c`
- `Rankings_DrawName`, `Rankings_DrawPassword` — owner-draw callbacks defined in `ui_rankings.c`

# code/q3_ui/ui_main.c
## File Purpose
This is the Q3 UI module's entry point for the QVM virtual machine. It implements `vmMain`, the sole gateway through which the engine dispatches commands into the UI module, and manages the registration and updating of all UI-related cvars.

## Core Responsibilities
- Expose `vmMain` as the single engine-facing entry point for all UI commands
- Route engine UI commands (init, shutdown, input events, refresh, menu activation) to the appropriate handler functions
- Declare all UI-side `vmCvar_t` globals that mirror engine cvars
- Define a `cvarTable_t` table mapping cvar structs to their name, default, and flags
- Implement `UI_RegisterCvars` and `UI_UpdateCvars` to batch-register and sync all cvars

## External Dependencies
- `ui_local.h` — aggregates `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`, all menu/subsystem declarations, and all `trap_*` syscall prototypes.
- `UI_Init`, `UI_Shutdown`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Refresh`, `UI_IsFullscreen`, `UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen` — defined in `ui_atoms.c` / other `q3_ui` files.
- `trap_Cvar_Register`, `trap_Cvar_Update` — defined in `ui_syscalls.c`; bridge to engine via QVM syscall ABI.
- `UI_API_VERSION` — defined as `4` in `ui_local.h` (overrides the value from `ui_public.h`).

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

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types)
- **Defined elsewhere:** `uis` (`uiStatic_t` global from `ui_atoms.c`), all `trap_*` syscall wrappers (`ui_syscalls.c`), `Menu_Draw`/`Menu_AddItem` (`ui_qmenu.c`), all `UI_*Menu()` navigation targets (their respective `.c` files), `color_red`/`menu_text_color`/`menu_null_sound` (`ui_qmenu.c`), `ui_cdkeychecked` (`ui_main.c`)

# code/q3_ui/ui_mfield.c
## File Purpose
Implements low-level editable text field widgets for the Q3 UI menu system. Provides both a raw `mfield_t` editing core and a higher-level `menufield_s` wrapper that integrates with the `menuframework_s` item system.

## Core Responsibilities
- Render a scrollable, optionally blinking text field with cursor (`MField_Draw`)
- Handle keyboard navigation: left/right arrows, Home, End, Delete, Insert (overstrike toggle)
- Handle character input with insert/overstrike modes and optional maxchars limit
- Clipboard paste via `trap_GetClipboardData`
- Initialize `menufield_s` bounding box geometry for hit-testing and layout
- Draw a `menufield_s` with focus highlight, label, and cursor arrow glyph
- Route menu-system key events to the underlying `mfield_t` with case/digit filtering

## External Dependencies
- **`ui_local.h`** — pulls in `mfield_t`, `menufield_s`, `menucommon_s`, key constants, draw style flags, `MAX_EDIT_LINE`, `QMF_*` flags, color externs.
- **Defined elsewhere:** `trap_GetClipboardData`, `trap_Key_GetOverstrikeMode`, `trap_Key_SetOverstrikeMode`, `trap_Key_IsDown`, `trap_Error`, `UI_DrawString`, `UI_DrawChar`, `UI_FillRect`, `Menu_ItemAtCursor`, `Q_islower`, `Q_isupper`, `Q_isalpha`, `menu_buzz_sound`, color arrays (`text_color_disabled`, `text_color_normal`, `text_color_highlight`, `listbar_color`).

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

## External Dependencies
- **`ui_local.h`** — menu framework types (`menuframework_s`, `menulist_s`, `menubitmap_s`, `menutext_s`, `menucommon_s`), all `trap_*` syscall declarations, `UI_PushMenu`/`UI_PopMenu`, `Menu_AddItem`, `Q_strncpyz`, `color_white`
- **Defined elsewhere:** `trap_Cvar_Set`, `trap_Cmd_ExecuteText`, `trap_FS_GetFileList`, `trap_R_RegisterShaderNoMip`, `trap_Print`, `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — all resolved through the VM syscall layer at runtime.

# code/q3_ui/ui_network.c
## File Purpose
Implements the Network Options menu screen within Quake III Arena's legacy UI module (q3_ui). It allows the player to configure their network data rate and navigate between the four System Setup sub-menus (Graphics, Display, Sound, Network).

## Core Responsibilities
- Declare and initialize all menu widgets for the Network Options screen
- Map the `rate` cvar's integer value to a human-readable connection-speed selection
- Write back the selected rate tier to the `rate` cvar on change
- Provide tab-like navigation to sibling option menus (Graphics, Display, Sound)
- Register/cache all required shader assets used by the menu
- Push the constructed menu onto the UI menu stack

## External Dependencies
- `ui_local.h` — pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types, and all `trap_*` syscall declarations
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip`, `color_white`, `color_red`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT`

# code/q3_ui/ui_options.c
## File Purpose
Implements the top-level "System Setup" options menu for Quake III Arena's legacy UI module. It presents four sub-menu navigation buttons (Graphics, Display, Sound, Network) plus a Back button, acting as a hub that dispatches to each specialized settings screen.

## Core Responsibilities
- Initialize and layout the System Setup menu (`optionsmenu_t`) with all UI items
- Pre-cache all artwork (frame bitmaps, back button) used by this menu
- Route activation events to the appropriate sub-menu or pop the menu stack
- Conditionally set fullscreen mode based on whether the client is already connected

## External Dependencies
- **Includes:** `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- **Defined elsewhere:**
  - `Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu` — `ui_qmenu.c` / `ui_atoms.c`
  - `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu` — respective `ui_video.c`, `ui_display.c`, `ui_sound.c`, `ui_network.c`
  - `trap_R_RegisterShaderNoMip`, `trap_GetClientState` — `ui_syscalls.c` (VM syscall wrappers)
  - `color_red`, `color_white` — `ui_atoms.c`

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

## External Dependencies
- `ui_local.h` — includes `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types, trap syscall declarations
- **Defined elsewhere:** `UI_DrawPlayer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo` (ui_players.c); `Menu_*` functions (ui_qmenu.c); all `trap_*` syscalls (ui_syscalls.c); `uis` global (ui_atoms.c); `menu_move_sound`, `menu_buzz_sound` (ui_qmenu.c)

# code/q3_ui/ui_players.c
## File Purpose
Implements the animated 3D player model preview rendering used in the Q3 UI (e.g., player selection screens). It manages model/skin/weapon loading, animation state machines for legs and torso, and submits all player-related render entities to the renderer each frame.

## Core Responsibilities
- Load and register player model parts (legs, torso, head), skins, and weapon models
- Parse `animation.cfg` files to populate the `animation_t` array
- Drive per-frame animation state machines for legs and torso (sequencing, blending, jump arcs)
- Compute hierarchical bone/tag placement for torso, head, gun, barrel, and muzzle flash entities
- Submit the full multi-part player entity (+ lights, sprite) to the renderer via `trap_R_*` syscalls
- Handle weapon-switch transitions, muzzle flash timing, and barrel spin for machine-gun-style weapons

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`
- **Defined elsewhere:** `bg_itemlist` (game item table), `uis` (global `uiStatic_t`), `weaponChangeSound`, all `trap_*` syscall wrappers, math utilities (`AnglesToAxis`, `MatrixMultiply`, `VectorMA`, etc.), animation constants (`LEGS_JUMP`, `TORSO_ATTACK`, `ANIM_TOGGLEBIT`, `MAX_ANIMATIONS`, etc.)

# code/q3_ui/ui_playersettings.c
## File Purpose
Implements the "Player Settings" menu screen in Quake III Arena's legacy UI module, allowing players to configure their in-game name, handicap level, and effects (rail trail) color, with a live 3D player model preview.

## Core Responsibilities
- Initialize and layout all widgets for the Player Settings menu
- Render custom owner-drawn controls: name field, handicap spinner, effects color picker, and animated player model
- Load current cvar values into UI controls on menu open (`PlayerSettings_SetMenuItems`)
- Persist UI state back to cvars on menu close or navigation (`PlayerSettings_SaveChanges`)
- Handle menu key events, routing escape/mouse2 to save-before-exit
- Preload/cache all required shader assets (`PlayerSettings_Cache`)
- Translate between UI color indices and game color codes via lookup tables

## External Dependencies
- `ui_local.h` — all menu framework types, draw utilities, trap syscalls, `playerInfo_t`, `uiStatic_t uis`
- **Defined elsewhere:** `Menu_AddItem`, `Menu_DefaultKey`, `UI_DrawPlayer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`, `UI_PushMenu`, `UI_PopMenu`, `UI_PlayerModelMenu`, `trap_Cvar_*`, `trap_R_RegisterShaderNoMip`, `trap_Key_GetOverstrikeMode`, `Q_strncpyz`, `Q_CleanStr`, `Q_IsColorString`, `Com_Clamp`, `g_color_table`, `color_white`, `text_color_normal`, `text_color_highlight`

# code/q3_ui/ui_preferences.c
## File Purpose
Implements the "Game Options" preferences menu for the Quake III Arena legacy UI (`q3_ui`). It allows the player to configure gameplay and visual cvars such as crosshair type, dynamic lights, wall marks, team overlay, and file downloading.

## Core Responsibilities
- Declare and initialize all widgets for the Game Options menu screen
- Read current cvar values into widget state on menu open (`Preferences_SetMenuItems`)
- Handle widget activation events and write changed values back to cvars (`Preferences_Event`)
- Provide a custom owner-draw function for the crosshair selector widget (`Crosshair_Draw`)
- Preload all required art assets and crosshair shaders (`Preferences_Cache`)
- Push the constructed menu onto the UI stack as the active screen (`UI_PreferencesMenu`)

## External Dependencies
- **`ui_local.h`** — menu framework types (`menuframework_s`, `menuradiobutton_s`, etc.), `trap_*` syscall wrappers, `UI_Push/PopMenu`, draw utilities
- **`trap_Cvar_VariableValue` / `trap_Cvar_SetValue` / `trap_Cvar_Reset`** — VM syscall layer (defined in `ui_syscalls.c`)
- **`trap_R_RegisterShaderNoMip`** — renderer syscall (defined in `ui_syscalls.c`)
- **`Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu`** — defined in `ui_atoms.c` / `ui_qmenu.c`
- **`Com_Clamp`** — defined in `game/q_shared.c`
- **cvars touched:** `cg_drawCrosshair`, `cg_simpleItems`, `cg_brassTime`, `cg_marks`, `cg_drawCrosshairNames`, `r_dynamiclight`, `r_fastsky`, `r_finish`, `cg_forcemodel`, `cg_drawTeamOverlay`, `cl_allowDownload`, `sv_allowDownload`

# code/q3_ui/ui_qmenu.c
## File Purpose
Implements the core menu framework and all standard widget types for Quake III Arena's legacy UI system (`q3_ui`). It provides initialization, drawing, and input handling for every interactive menu element, plus the top-level menu management routines.

## Core Responsibilities
- Register and cache all shared UI assets (shaders, sounds) via `Menu_Cache`
- Initialize widget bounding boxes and state on `Menu_AddItem`
- Dispatch per-frame drawing for all widget types via `Menu_Draw`
- Route keyboard/mouse input to the focused widget via `Menu_DefaultKey`
- Manage menu cursor movement, focus transitions, and wrap-around via `Menu_AdjustCursor` / `Menu_CursorMoved`
- Provide sound feedback (move, buzz, in/out) for all interactive events
- Support a debug overlay (bounding-box visualization) under `#ifndef NDEBUG`

## External Dependencies
- **`ui_local.h`** — brings in all widget type definitions, flag constants, `uis` global, and `trap_*` syscall declarations.
- **`trap_R_RegisterShaderNoMip`, `trap_R_SetColor`, `trap_S_RegisterSound`, `trap_S_StartLocalSound`** — renderer/audio syscalls, defined in `ui_syscalls.c`.
- **`UI_Draw*`, `UI_FillRect`, `UI_SetColor`, `UI_CursorInRect`** — defined in `ui_atoms.c`.
- **`MenuField_Init`, `MenuField_Draw`, `MenuField_Key`** — defined in `ui_mfield.c`.
- **`UI_PopMenu`** — defined in `ui_atoms.c`.
- **`uis`** (`uiStatic_t`) — singleton global defined in `ui_atoms.c`.
- **`Menu_ItemAtCursor`** — defined in this file; also declared `extern` in `ui_local.h` for use by other modules.

# code/q3_ui/ui_rankings.c
## File Purpose
Implements the in-game "Rankings" overlay menu for Quake III Arena's online ranking system (GRank). It presents context-sensitive options (login, logout, sign up, spectate, setup, leave arena) based on the player's current ranking status.

## Core Responsibilities
- Initialize and display the rankings popup menu with a decorative frame
- Show/hide/gray out menu items dynamically based on `client_status` cvar (grank status)
- Route menu events to appropriate UI screens or game commands
- Provide custom field draw helpers for name and password input fields (used by login/signup menus)
- Pre-cache the frame shader asset

## External Dependencies
- `ui_local.h` — pulls in all menu framework types, trap syscalls, color tables, and UI helper declarations
- **Defined elsewhere:** `grank_status_t`, `QGR_STATUS_*` constants (GRank headers), `trap_CL_UI_RankUserRequestLogout`, `UI_LoginMenu`, `UI_SignupMenu`, `UI_SetupMenu`, `UI_ForceMenuOff`, `UI_PushMenu`, `Menu_AddItem`, `UI_DrawChar`, `trap_Key_GetOverstrikeMode`, `Q_CleanStr`, `Q_strncpyz`, `g_color_table`, `ColorIndex`, `color_white`, `text_color_normal`, `text_color_highlight`

# code/q3_ui/ui_rankstatus.c
## File Purpose
Implements a modal status dialog for the GRank (Global Ranking) online ranking system, displaying error or result messages when a ranking operation completes. It maps `grank_status_t` codes to human-readable strings and routes the user to appropriate follow-up menus on dismissal.

## Core Responsibilities
- Read `client_status` cvar to determine the current `grank_status_t` code
- Map ranking status codes to display strings (e.g., "Invalid password", "Timed out")
- Build and display a simple two-item menu: a static message and an OK button
- On OK, pop this menu and push the appropriate follow-up menu (rankings, login, signup) based on the original status code
- Early-exit silently for benign statuses (`QGR_STATUS_NEW`, `QGR_STATUS_PENDING`, `QGR_STATUS_SPECTATOR`, `QGR_STATUS_ACTIVE`)
- Pre-cache the frame shader via `RankStatus_Cache`

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, menu framework types, all `trap_*` syscall declarations)
- **Defined elsewhere:**
  - `grank_status_t` and its `QGR_STATUS_*` constants — ranking system types (defined in ranking headers pulled through `ui_local.h`)
  - `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip`, `trap_CL_UI_RankUserReset` — VM syscall stubs (`ui_syscalls.c`)
  - `UI_PopMenu`, `UI_PushMenu`, `UI_ForceMenuOff` — menu stack management (`ui_atoms.c`)
  - `UI_RankingsMenu`, `UI_LoginMenu`, `UI_SignupMenu` — sibling ranking UI screens
  - `Menu_AddItem` — menu framework (`ui_qmenu.c`)
  - `colorRed` — shared color constant (`ui_qmenu.c` / `q_shared.c`)

# code/q3_ui/ui_removebots.c
## File Purpose
Implements the in-game "Remove Bots" menu for Quake III Arena's legacy UI module. It allows a human player to view currently connected bot clients and kick one by client number via a console command.

## Core Responsibilities
- Enumerate active bot clients from server config strings by checking for a non-zero `skill` field
- Display up to 7 bot names in a scrollable list
- Track which bot entry is selected and visually distinguish it (orange vs. white color)
- Issue a `clientkick <num>` command when the user activates the Delete button
- Register and cache all required artwork shaders on demand
- Push/pop the menu onto the UI menu stack

## External Dependencies
- `ui_local.h` — all menu framework types, widget types, trap syscall declarations, color vectors
- `trap_GetConfigString` — reads `CS_SERVERINFO` and `CS_PLAYERS + n` (defined in engine/syscall layer)
- `trap_Cmd_ExecuteText` — issues console commands to the engine
- `trap_R_RegisterShaderNoMip` — registers 2D art assets
- `Info_ValueForKey`, `Q_strncpyz`, `Q_CleanStr` — defined in `q_shared.c`
- `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — defined in `ui_atoms.c` / `ui_qmenu.c`
- `MAX_BOTS` — defined in `bg_public.h`

# code/q3_ui/ui_saveconfig.c
## File Purpose
Implements the "Save Config" menu screen for Quake III Arena's legacy UI module (`q3_ui`). It presents a full-screen dialog allowing the player to type a filename and write the current game configuration to a `.cfg` file via a console command.

## Core Responsibilities
- Initialize and layout the Save Config menu widgets (banner, background, text field, back/save buttons)
- Pre-cache all bitmap art assets used by the menu
- Handle the "Back" button event by popping the menu stack
- Handle the "Save" button event by stripping the file extension and dispatching a `writeconfig` command
- Provide a custom owner-draw callback for the filename input field
- Expose the menu entry point (`UI_SaveConfigMenu`) and asset cache function to the rest of the UI module

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_ItemAtCursor` — `ui_qmenu.c` / `ui_atoms.c`
  - `MField_Draw` — `ui_mfield.c`
  - `UI_DrawProportionalString`, `UI_FillRect` — `ui_atoms.c`
  - `trap_R_RegisterShaderNoMip`, `trap_Cmd_ExecuteText` — `ui_syscalls.c` (VM syscall wrappers)
  - `COM_StripExtension` — `q_shared.c`
  - `va` — `q_shared.c`
  - Color constants (`color_orange`, `colorBlack`, `colorRed`, `text_color_highlight`) — `ui_qmenu.c`

# code/q3_ui/ui_serverinfo.c
## File Purpose
Implements the "Server Info" UI menu in Quake III Arena's legacy q3_ui module. It displays key-value pairs from the current server's config string and provides "Add to Favorites" and "Back" actions.

## Core Responsibilities
- Fetch and display the server's `CS_SERVERINFO` config string as a key-value table
- Vertically center the info table based on the number of lines
- Allow the player to add the current server to the favorites list (cvars `server1`–`serverN`)
- Prevent the "Add to Favorites" action when a local server is running (`sv_running`)
- Pre-cache UI art assets via `trap_R_RegisterShaderNoMip`
- Provide keyboard and mouse event routing through the standard menu framework

## External Dependencies
- `ui_local.h` — menu framework types, trap functions, draw utilities, `MAX_FAVORITESERVERS`
- `trap_GetConfigString` / `CS_SERVERINFO` — defined in engine/qcommon layer
- `Info_NextPair` — defined in `q_shared.c`
- `UI_PushMenu`, `UI_PopMenu`, `Menu_Draw`, `Menu_DefaultKey`, `Menu_AddItem` — defined in `ui_qmenu.c`
- `UI_DrawString` — defined in `ui_atoms.c`
- `trap_R_RegisterShaderNoMip`, `trap_Cvar_*` — syscall stubs defined in `ui_syscalls.c`

# code/q3_ui/ui_servers2.c
## File Purpose
Implements the Quake III Arena multiplayer server browser menu ("Arena Servers"), handling server discovery, ping querying, filtering, sorting, and connection initiation. It manages four server source types: Local, Internet (Global), MPlayer, and Favorites.

## Core Responsibilities
- Initialize and render the server browser menu with all UI controls
- Manage ping request queues to discover and measure server latency
- Filter server list by game type, full/empty status, and max ping
- Sort server list by hostname, map, open slots, game type, or ping
- Persist and load favorite server addresses via cvars (`server1`–`server16`)
- Handle PunkBuster enable/disable confirmation dialogs
- Connect to a selected server via `connect` command

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h`, all menu framework types and trap syscalls
- **Defined elsewhere:** `trap_LAN_*` (server list and ping syscalls), `trap_Cmd_ExecuteText`, `trap_Cvar_*`, `trap_R_RegisterShaderNoMip`, `Menu_Draw`, `Menu_AddItem`, `Menu_DefaultKey`, `ScrollList_Key`, `UI_PushMenu`, `UI_PopMenu`, `UI_ConfirmMenu_Style`, `UI_SpecifyServerMenu`, `UI_StartServerMenu`, `UI_Message`, `uis` (global UI state), `qsort` (libc)

# code/q3_ui/ui_setup.c
## File Purpose
Implements the Setup menu screen for Quake III Arena's legacy UI module (`q3_ui`). It presents a top-level configuration hub from which players navigate to sub-menus covering player settings, controls, graphics, game options, and CD key entry.

## Core Responsibilities
- Define and initialize all menu items for the Setup screen layout
- Route menu item activation events to their respective sub-menu functions
- Conditionally show the "DEFAULTS" option only when not in-game (i.e., `cl_paused == 0`)
- Confirm and execute a full configuration reset via `exec default.cfg` / `cvar_restart` / `vid_restart`
- Pre-cache all bitmap assets used by the Setup screen
- Expose public entry points (`UI_SetupMenu`, `UI_SetupMenu_Cache`) for the broader UI system

## External Dependencies
- **Includes:** `ui_local.h` (aggregates `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework declarations, and all trap syscall prototypes)
- **Defined elsewhere:**
  - `UI_PlayerSettingsMenu`, `UI_ControlsMenu`, `UI_GraphicsOptionsMenu`, `UI_PreferencesMenu`, `UI_CDKeyMenu` — sub-menu entry points in their respective `.c` files
  - `UI_ConfirmMenu` — `ui_confirm.c`
  - `UI_PushMenu`, `UI_PopMenu` — `ui_atoms.c`
  - `Menu_AddItem`, `Menu_Draw` — `ui_qmenu.c`
  - `trap_*` syscalls — `ui_syscalls.c` (VM trap layer)
  - `color_white`, `color_red`, `color_yellow` — `ui_atoms.c`

# code/q3_ui/ui_signup.c
## File Purpose
Implements the user account sign-up menu for Quake III Arena's GRank (Global Rankings) online ranking system. It provides a form UI for new players to register a ranked account by supplying a name, password (with confirmation), and email address.

## Core Responsibilities
- Define and initialize all UI widgets for the sign-up form (labels, input fields, buttons)
- Validate that the password and confirmation fields match before submission
- Invoke `trap_CL_UI_RankUserCreate` to submit registration data to the rankings backend
- Conditionally disable all input fields if the player's `client_status` indicates they are not eligible to sign up (i.e., already registered)
- Preload the frame bitmap asset via `Signup_Cache`
- Push the initialized menu onto the UI menu stack via `UI_SignupMenu`

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types, and all trap declarations)
- **Defined elsewhere:**
  - `trap_CL_UI_RankUserCreate` — ranking system syscall, not declared in the bundled header (GRank-specific extension)
  - `Rankings_DrawName`, `Rankings_DrawPassword`, `Rankings_DrawText` — ownerdraw callbacks defined in `ui_rankings.c`
  - `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` — defined in `ui_atoms.c`
  - `grank_status_t`, `QGR_STATUS_NEW`, `QGR_STATUS_SPECTATOR` — defined in GRank headers (not shown)
  - `Menu_AddItem` — defined in `ui_qmenu.c`

# code/q3_ui/ui_sound.c
## File Purpose
Implements the Sound Options menu screen within Quake III Arena's legacy UI module (`q3_ui`). It allows players to configure effects volume, music volume, and sound quality (sample rate/compression) through a standard menu framework.

## Core Responsibilities
- Initialize and lay out all sound options menu widgets (sliders, spin control, navigation tabs, decorative bitmaps)
- Read current sound CVars (`s_volume`, `s_musicvolume`, `s_compression`) to populate widget state on open
- Write CVar changes back to the engine when the user adjusts controls
- Navigate to sibling option menus (Graphics, Display, Network) or go back
- Trigger `snd_restart` when sound quality is changed, requiring a sound system reload

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types and prototypes)
- **Defined elsewhere:**
  - `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip` — syscall wrappers in `ui_syscalls.c`
  - `UI_PopMenu`, `UI_PushMenu`, `UI_ForceMenuOff`, `Menu_AddItem`, `Menu_SetCursorToItem` — menu framework in `ui_atoms.c` / `ui_qmenu.c`
  - `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_NetworkOptionsMenu` — sibling option menu files
  - `color_white`, `color_red`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT` — shared UI constants

# code/q3_ui/ui_sparena.c
## File Purpose
Handles the launch sequence for a single-player arena in Quake III Arena's UI layer. It configures the necessary CVars and issues the server command to start a specific SP map.

## Core Responsibilities
- Ensures `sv_maxclients` is at least 8 before starting an SP arena
- Resolves the numeric SP level index from arena metadata, with special-case handling for "training" and "final" arenas
- Writes the resolved level selection into the `ui_spSelection` CVar for downstream use
- Executes the `spmap` command to load the chosen map

## External Dependencies
- **Includes:** `ui_local.h` (which pulls in `q_shared.h`, `bg_public.h`, trap syscall declarations)
- **Defined elsewhere:**
  - `trap_Cvar_VariableValue`, `trap_Cvar_SetValue`, `trap_Cmd_ExecuteText` — UI syscall stubs (`ui_syscalls.c`)
  - `Info_ValueForKey`, `Q_stricmp`, `atoi`, `va` — shared utilities (`q_shared.c`)
  - `UI_GetNumSPTiers`, `ARENAS_PER_TIER` — SP game info module (`ui_gameinfo.c` / `bg_public.h`)

# code/q3_ui/ui_specifyleague.c
## File Purpose
Implements the "Specify League" UI menu for Quake III Arena's Global Rankings system, allowing players to enter a username, query available leagues for that player, and select one to set as the active `sv_leagueName` cvar.

## Core Responsibilities
- Initialize and lay out the Specify League menu screen with decorative bitmaps, a player name text field, a scrollable league list, and navigation buttons
- Query the Global Rankings backend for leagues associated with a given player name via `trap_CL_UI_RankGetLeauges`
- Populate a fixed-size list box with league names retrieved from numbered cvars (`leaguename1`, `leaguename2`, …)
- Re-query the league list when the player name field loses focus and the name has changed
- Write the selected league name to `sv_leagueName` cvar on back/confirm
- Pre-cache all required UI art shaders via `SpecifyLeague_Cache`

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- `trap_CL_UI_RankGetLeauges` — Global Rankings syscall, defined in `ui_syscalls.c` / engine; not declared in the bundled header (likely a GRank extension)
- `trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer`, `trap_R_RegisterShaderNoMip` — engine syscalls declared in `ui_local.h`
- `Menu_AddItem`, `ScrollList_Key`, `UI_PushMenu`, `UI_PopMenu` — defined in `ui_qmenu.c` / `ui_atoms.c`
- `Q_strncpyz`, `Q_strncmp`, `va` — string utilities from `q_shared.c`

# code/q3_ui/ui_specifyserver.c
## File Purpose
Implements the "Specify Server" UI menu, allowing players to manually enter a server IP address and port number to connect to directly. It is a simple two-field input form within the Q3 legacy UI module.

## Core Responsibilities
- Define and initialize all menu items (banner, decorative frames, address/port fields, go/back buttons)
- Handle user activation events for "Go" (connect) and "Back" (pop menu) buttons
- Preload/cache all required bitmap art assets via the renderer
- Build and dispatch the `connect <address>:<port>` command string to the engine
- Push the assembled menu onto the active UI menu stack

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types/macros)
- **Defined elsewhere:** `Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip`, `Com_sprintf`, `va`, `color_white` — all provided by the broader Q3 UI/engine runtime

# code/q3_ui/ui_splevel.c
## File Purpose
Implements the single-player level selection menu for Quake III Arena, allowing players to browse tier-based arena sets, select maps, view completion status, and navigate to the skill selection screen.

## Core Responsibilities
- Initialize and layout the level select menu with up to 4 level thumbnail bitmaps per tier
- Handle tier navigation via left/right arrow buttons
- Display player icon, awards/medals, and bot opponent portraits
- Track and display level completion status with skill-rated completion images
- Handle special-case tiers (training and final) with single-map display
- Provide reset-game confirmation flow and custom/skirmish navigation

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `tr_types.h`, `ui_public.h`
- **Defined elsewhere:** `UI_GetArenaInfoByNumber`, `UI_GetSpecialArenaInfo`, `UI_GetBotInfoByName`, `UI_GetBestScore`, `UI_GetAwardLevel`, `UI_GetCurrentGame`, `UI_NewGame`, `UI_GetNumSPTiers`, `UI_GetNumSPArenas`, `UI_SPSkillMenu`, `UI_StartServerMenu`, `UI_PlayerSettingsMenu`, `UI_ConfirmMenu`, `Menu_Draw`, `Menu_AddItem`, `Bitmap_Init`, `Menu_SetCursorToItem`, `UI_PushMenu`, `UI_PopMenu`, all `trap_*` syscalls, `uis` global, `ui_medalPicNames[]`, `ui_medalSounds[]`, `ARENAS_PER_TIER`, `PULSE_DIVISOR`

# code/q3_ui/ui_sppostgame.c
## File Purpose
Implements the single-player postgame menu for Quake III Arena, displayed after a match ends. It orchestrates a three-phase animated sequence: podium presentation, award medal display, then interactive buttons for replay/next/menu navigation.

## Core Responsibilities
- Parse postgame command arguments (scores, ranks, award stats) into menu state
- Drive a three-phase timed presentation (podium → awards → navigation buttons)
- Display and animate per-award medals with sounds
- Evaluate tier/level progression logic to determine the "Next" level destination
- Trigger tier cinematic videos upon tier completion
- Persist best scores and award data via `UI_SetBestScore` / `UI_LogAwardData`
- Register and play winner/loser music and announcement sounds

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, `keycodes.h`, all menu/trap declarations
- **Defined elsewhere:** `UI_GetArenaInfoByMap`, `UI_GetArenaInfoByNumber`, `UI_TierCompleted`, `UI_ShowTierVideo`, `UI_GetNumSPTiers`, `UI_GetSpecialArenaInfo`, `UI_SetBestScore`, `UI_LogAwardData`, `UI_GetAwardLevel`, `UI_GetCurrentGame`, `UI_SPArena_Start`, `Menu_*` functions, all `trap_*` syscalls, `uis` global, draw utilities (`UI_DrawProportionalString`, `UI_DrawNamedPic`, `UI_DrawString`)

# code/q3_ui/ui_spreset.c
## File Purpose
Implements the single-player "Reset Game" confirmation dialog for Quake III Arena's UI module. It presents a YES/NO prompt to the player and, on confirmation, wipes all single-player progress data and restarts the level menu from the beginning.

## Core Responsibilities
- Renders the reset confirmation dialog with a decorative frame and warning text
- Handles YES/NO menu item selection via mouse and keyboard (including `Y`/`N` hotkeys)
- On confirmation: calls `UI_NewGame()`, resets `ui_spSelection` to 0, pops the current menu stack entries, and re-launches the SP level menu
- Caches the background frame shader on demand
- Positions the `YES / NO` text layout dynamically using proportional string width calculations
- Sets fullscreen vs. overlay mode based on whether a game session is currently connected

## External Dependencies
- `ui_local.h` — pulls in all UI framework types, menu item types, trap syscalls, draw utilities, and SP game info functions
- **Defined elsewhere:** `UI_NewGame` (`ui_gameinfo.c`), `UI_SPLevelMenu` (`ui_spLevel.c`), `UI_PopMenu` / `UI_PushMenu` / `UI_DrawNamedPic` / `UI_DrawProportionalString` / `UI_ProportionalStringWidth` (`ui_atoms.c`), `Menu_Draw` / `Menu_DefaultKey` / `Menu_AddItem` / `Menu_SetCursorToItem` (`ui_qmenu.c`), `trap_*` syscall wrappers (`ui_syscalls.c`), `trap_R_RegisterShaderNoMip` (renderer via VM syscall)

# code/q3_ui/ui_spskill.c
## File Purpose
Implements the single-player difficulty selection menu in Quake III Arena's UI module. It presents five skill levels ("I Can Win" through "NIGHTMARE!"), persists the selection to the `g_spSkill` cvar, and transitions into the arena start flow.

## Core Responsibilities
- Initialize and lay out all menu widgets for the skill selection screen
- Highlight the currently selected skill in white; all others in red
- Update `g_spSkill` cvar when the player selects a difficulty
- Swap the displayed skill-level illustration (`art_skillPic`) on selection change
- Play a special sound for NIGHTMARE difficulty; silence sound otherwise
- Navigate back to the previous menu or forward to `UI_SPArena_Start`
- Pre-cache all shaders and sounds required by this menu

## External Dependencies
- `ui_local.h` — menu framework types, widget types, trap syscall declarations, helper functions
- **Defined elsewhere:** `UI_SPArena_Start`, `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `Menu_DefaultKey`, `Com_Clamp`, all `trap_*` syscall wrappers, `color_red`, `color_white`

# code/q3_ui/ui_startserver.c
## File Purpose
Implements three interconnected UI menus for launching a multiplayer or single-player server: the **Start Server** map-selection menu, the **Server Options** configuration menu, and the **Bot Select** picker menu. Together they form a wizard-style flow: pick a map → configure options/bots → execute the server launch.

## Core Responsibilities
- Display a paginated grid of level-shot thumbnails filtered by game type for map selection
- Allow game type selection (FFA, Team DM, Tournament, CTF) and re-filter the map list accordingly
- Provide server configuration controls: frag/time/capture limits, friendly fire, pure server, dedicated mode, hostname, bot skill, PunkBuster
- Manage up to 12 player slots as Open/Bot/Closed with optional team assignment
- Display a paginated bot portrait grid for bot selection, sorted alphabetically
- Build and execute the `map` command along with `addbot` and `team` commands to start the server

## External Dependencies
- **Includes:** `ui_local.h` (menu framework, trap syscalls, shared types)
- **Defined elsewhere:** `punkbuster_items[]` (extern from `ui_servers2.c`); `UI_ServerOptionsMenu` forward-declared static but called from `StartServer_MenuEvent`
- **Trap syscalls used:** `trap_R_RegisterShaderNoMip`, `trap_Cvar_SetValue`, `trap_Cvar_Set`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cmd_ExecuteText`
- **UI info queries:** `UI_GetNumArenas`, `UI_GetArenaInfoByNumber`, `UI_GetArenaInfoByMap`, `UI_GetNumBots`, `UI_GetBotInfoByNumber`, `UI_GetBotInfoByName`, `Info_ValueForKey`

# code/q3_ui/ui_team.c
## File Purpose
Implements the in-game Team Selection overlay menu for Quake III Arena, allowing players to join the red team, blue team, free-for-all, or spectate. Menu items are conditionally grayed out based on the current server game type.

## Core Responsibilities
- Define and initialize the team selection menu (`s_teammain`)
- Register the decorative frame shader asset via cache call
- Handle menu item activation events by sending server commands (`cmd team red/blue/free/spectator`)
- Query `CS_SERVERINFO` to determine current game type and disable irrelevant options
- Push the initialized menu onto the UI menu stack

## External Dependencies
- `ui_local.h` — pulls in `menuframework_s`, `menubitmap_s`, `menutext_s`, `menucommon_s`, `QM_ACTIVATED`, `QMF_*` flags, `MTYPE_*` constants, game type enums (`GT_TEAM`, `GT_CTF`, etc.), `CS_SERVERINFO`, and all `trap_*` / `UI_*` function declarations
- `trap_Cmd_ExecuteText` — defined in `ui_syscalls.c`, bridges to engine
- `trap_GetConfigString` — defined in `ui_syscalls.c`, bridges to engine
- `trap_R_RegisterShaderNoMip` — defined in `ui_syscalls.c`, bridges to renderer
- `Info_ValueForKey` — defined in `q_shared.c`
- `UI_ForceMenuOff`, `UI_PushMenu`, `Menu_AddItem` — defined in `ui_atoms.c` / `ui_qmenu.c`

# code/q3_ui/ui_teamorders.c
## File Purpose
Implements the in-game Team Orders menu for Quake III Arena, allowing players to issue commands to bot teammates. It presents a two-step selection UI: first choose a bot (or "Everyone"), then choose an order, which is transmitted as a `say_team` chat message.

## Core Responsibilities
- Build a dynamic list of bot teammates from server config strings
- Render a scrollable, owner-drawn proportional-font list widget
- Handle two-phase selection: bot target → order message
- Format and dispatch `say_team` commands with the selected bot name interpolated
- Guard menu access (team game only, non-spectators)
- Pre-cache required artwork shaders

## External Dependencies
- **Includes:** `ui_local.h` → pulls in `q_shared.h`, `bg_public.h`, UI framework types and trap declarations
- **Defined elsewhere:** `trap_GetClientState`, `trap_GetConfigString`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip`, `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem`, `Menu_ItemAtCursor`, `Menu_DefaultKey`, `UI_DrawProportionalString`, `UI_CursorInRect`, `Com_sprintf`, `va`, `Q_strncpyz`, `Q_CleanStr`, `Info_ValueForKey`, `uis` (global UI state), game constants `GT_CTF`, `GT_TEAM`, `TEAM_SPECTATOR`, `CS_SERVERINFO`, `CS_PLAYERS`

# code/q3_ui/ui_video.c
## File Purpose
Implements two UI menus for Quake III Arena: the **Driver Info** screen (read-only display of OpenGL vendor/renderer/extension strings) and the **Graphics Options** screen (interactive controls for video settings such as resolution, color depth, texture quality, and geometry detail).

## Core Responsibilities
- Build and display the Driver Info menu, parsing and rendering GL extension strings in two columns
- Build and display the Graphics Options menu with spin controls, sliders, and bitmaps for all major renderer cvars
- Apply pending video changes by writing renderer cvars and issuing `vid_restart`
- Track initial video state (`s_ivo`) to determine when the "Apply" button should be shown
- Match current settings against predefined quality presets (High/Normal/Fast/Fastest/Custom)
- Navigate between sibling option menus (Display, Sound, Network) via tab-style text buttons
- Preload all UI art shaders via cache functions

## External Dependencies
- `ui_local.h` — menu framework types, `uis` global (`uiStatic_t`), all `trap_*` syscalls, `UI_Push/PopMenu`, `UI_DrawString`, color constants
- `uis.glconfig` (`glconfig_t`) — GL vendor/renderer/version/extensions strings, driver type, hardware type, color/depth/stencil bits
- External menu functions: `Menu_Draw`, `Menu_AddItem`, `Menu_SetCursorToItem`
- External navigation targets (defined elsewhere): `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`
- Renderer cvars written: `r_mode`, `r_fullscreen`, `r_colorbits`, `r_depthbits`, `r_stencilbits`, `r_texturebits`, `r_picmip`, `r_vertexLight`, `r_lodBias`, `r_subdivisions`, `r_textureMode`, `r_allowExtensions`, `r_glDriver`
- `OPENGL_DRIVER_NAME`, `_3DFX_DRIVER_NAME` — defined elsewhere (platform headers)

