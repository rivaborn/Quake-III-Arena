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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `playersettings_t` | struct | Aggregate of all menu widgets, cached shader handles, live `playerInfo_t`, current effects index, and last-known model string |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_playersettings` | `playersettings_t` | static (file) | Sole instance of the player settings menu; persists for the menu's lifetime |
| `gamecodetoui[]` | `int[7]` | static (file) | Maps game `color1` cvar values (1-based) to UI spin-control indices |
| `uitogamecode[]` | `int[7]` | static (file) | Reverse map: UI spin index → game color code written to `color1` |
| `handicap_items[]` | `const char*[]` | static (file) | NULL-terminated string list for the handicap spin control display |

## Key Functions / Methods

### PlayerSettings_DrawName
- **Signature:** `static void PlayerSettings_DrawName( void *self )`
- **Purpose:** Owner-draw callback for the name text field; renders the label, the raw buffer with color codes parsed, a blink cursor when focused, and a clean big-font preview at the bottom of the screen.
- **Inputs:** `self` — pointer to the `menufield_s` for the name widget
- **Outputs/Return:** void
- **Side effects:** Issues UI draw calls
- **Calls:** `UI_DrawProportionalString`, `Q_IsColorString`, `UI_DrawChar`, `trap_Key_GetOverstrikeMode`, `Q_strncpyz`, `Q_CleanStr`
- **Notes:** Skips color-code parsing when the field has focus so the user sees raw escape sequences while editing.

### PlayerSettings_DrawHandicap
- **Signature:** `static void PlayerSettings_DrawHandicap( void *self )`
- **Purpose:** Owner-draw callback for the handicap spin control; renders label and current value string.
- **Inputs:** `self` — `menulist_s*`
- **Outputs/Return:** void
- **Side effects:** UI draw calls
- **Calls:** `UI_DrawProportionalString`

### PlayerSettings_DrawEffects
- **Signature:** `static void PlayerSettings_DrawEffects( void *self )`
- **Purpose:** Owner-draw callback for the effects (rail color) spin; renders label, a base-bar shader, and a highlight pip shader at the selected index.
- **Inputs:** `self` — `menulist_s*`
- **Outputs/Return:** void
- **Side effects:** UI draw calls
- **Calls:** `UI_DrawProportionalString`, `UI_DrawHandlePic`

### PlayerSettings_DrawPlayer
- **Signature:** `static void PlayerSettings_DrawPlayer( void *self )`
- **Purpose:** Owner-draw callback for the player preview bitmap; detects model cvar changes and re-initializes `playerInfo_t` before calling the player renderer.
- **Inputs:** `self` — `menubitmap_s*`
- **Outputs/Return:** void
- **Side effects:** Reads `model` cvar; mutates `s_playersettings.playerinfo` and `s_playersettings.playerModel` on model change
- **Calls:** `trap_Cvar_VariableStringBuffer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`, `UI_DrawPlayer`

### PlayerSettings_SaveChanges
- **Signature:** `static void PlayerSettings_SaveChanges( void )`
- **Purpose:** Flushes all current UI widget values to their corresponding cvars (`name`, `handicap`, `color1`).
- **Inputs:** None (reads from `s_playersettings`)
- **Outputs/Return:** void
- **Side effects:** Sets cvars via `trap_Cvar_Set` / `trap_Cvar_SetValue`
- **Calls:** `trap_Cvar_Set`, `trap_Cvar_SetValue`

### PlayerSettings_MenuKey
- **Signature:** `static sfxHandle_t PlayerSettings_MenuKey( int key )`
- **Purpose:** Intercepts K_MOUSE2/K_ESCAPE to trigger `PlayerSettings_SaveChanges` before delegating to the default key handler.
- **Inputs:** `key` — raw key code
- **Outputs/Return:** `sfxHandle_t` from `Menu_DefaultKey`
- **Side effects:** May save cvars
- **Calls:** `PlayerSettings_SaveChanges`, `Menu_DefaultKey`

### PlayerSettings_SetMenuItems
- **Signature:** `static void PlayerSettings_SetMenuItems( void )`
- **Purpose:** Reads current cvar values and populates all widget state fields before the menu is shown.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Mutates `s_playersettings` widget fields; calls `UI_PlayerInfo_SetModel`/`SetInfo`
- **Calls:** `UI_Cvar_VariableString`, `trap_Cvar_VariableValue`, `Com_Clamp`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`

### PlayerSettings_MenuEvent
- **Signature:** `static void PlayerSettings_MenuEvent( void *ptr, int event )`
- **Purpose:** Widget activation callback; handles ID_HANDICAP live-write, ID_MODEL → navigate to model picker, ID_BACK → save and pop.
- **Inputs:** `ptr` — `menucommon_s*`; `event` — QM_* constant
- **Side effects:** May set cvars, push/pop menu stack
- **Calls:** `trap_Cvar_Set`, `PlayerSettings_SaveChanges`, `UI_PlayerModelMenu`, `UI_PopMenu`

### PlayerSettings_MenuInit
- **Signature:** `static void PlayerSettings_MenuInit( void )`
- **Purpose:** Zeroes state, calls `PlayerSettings_Cache`, constructs and registers all menu items, then calls `PlayerSettings_SetMenuItems`.
- **Calls:** `PlayerSettings_Cache`, `Menu_AddItem`, `PlayerSettings_SetMenuItems`

### PlayerSettings_Cache
- **Signature:** `void PlayerSettings_Cache( void )`
- **Purpose:** Registers all UI art shaders with the renderer; stores `fxBasePic` and `fxPic[7]` handles into `s_playersettings`.
- **Side effects:** Renderer shader registration
- **Calls:** `trap_R_RegisterShaderNoMip`

### UI_PlayerSettingsMenu
- **Signature:** `void UI_PlayerSettingsMenu( void )`
- **Purpose:** Public entry point; inits the menu and pushes it onto the menu stack.
- **Calls:** `PlayerSettings_MenuInit`, `UI_PushMenu`

## Control Flow Notes
Called from the main/setup menu hierarchy at any point. `UI_PlayerSettingsMenu` → `PlayerSettings_MenuInit` → `UI_PushMenu` makes it the active menu. Each frame the UI system calls the menu's draw pipeline, which invokes the owner-draw callbacks. On deactivation (back/escape), `PlayerSettings_SaveChanges` is called before `UI_PopMenu`. The player model preview updates lazily per-frame by diffing the `model` cvar against a cached string.

## External Dependencies
- `ui_local.h` — all menu framework types, draw utilities, trap syscalls, `playerInfo_t`, `uiStatic_t uis`
- **Defined elsewhere:** `Menu_AddItem`, `Menu_DefaultKey`, `UI_DrawPlayer`, `UI_PlayerInfo_SetModel`, `UI_PlayerInfo_SetInfo`, `UI_PushMenu`, `UI_PopMenu`, `UI_PlayerModelMenu`, `trap_Cvar_*`, `trap_R_RegisterShaderNoMip`, `trap_Key_GetOverstrikeMode`, `Q_strncpyz`, `Q_CleanStr`, `Q_IsColorString`, `Com_Clamp`, `g_color_table`, `color_white`, `text_color_normal`, `text_color_highlight`
