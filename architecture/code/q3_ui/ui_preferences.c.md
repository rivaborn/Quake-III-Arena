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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `preferences_t` | struct | Aggregates all menu widgets (banner, frames, radio buttons, lists, bitmaps) and the 10 crosshair shader handles for the Game Options screen |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_preferences` | `preferences_t` | static (file) | Sole instance of the preferences menu; persists for the lifetime of the menu |
| `teamoverlay_names` | `const char *[]` | static (file) | String table for the "Draw Team Overlay" spin-control: off / upper right / lower right / lower left |

## Key Functions / Methods

### Preferences_SetMenuItems
- **Signature:** `static void Preferences_SetMenuItems( void )`
- **Purpose:** Reads live cvar values and populates `s_preferences` widget `curvalue` fields so UI state matches engine state on menu open.
- **Inputs:** None (reads from engine cvars via `trap_Cvar_VariableValue`)
- **Outputs/Return:** void
- **Side effects:** Mutates `s_preferences` widget fields
- **Calls:** `trap_Cvar_VariableValue`, `Com_Clamp`
- **Notes:** `r_fastsky` is inverted — `highqualitysky.curvalue = (r_fastsky == 0)`.

### Preferences_Event
- **Signature:** `static void Preferences_Event( void* ptr, int notification )`
- **Purpose:** Callback for all menu items; writes the updated widget value back to the corresponding cvar when a widget is activated.
- **Inputs:** `ptr` — pointer to the activated `menucommon_s`; `notification` — event type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** void
- **Side effects:** Calls `trap_Cvar_SetValue` or `trap_Cvar_Reset`; `ID_ALLOWDOWNLOAD` writes both `cl_allowDownload` and `sv_allowDownload`; `ID_BACK` calls `UI_PopMenu`
- **Calls:** `trap_Cvar_SetValue`, `trap_Cvar_Reset`, `UI_PopMenu`
- **Notes:** `ID_EJECTINGBRASS` resets to default (non-zero) rather than using a hardcoded value when enabling.

### Crosshair_Draw
- **Signature:** `static void Crosshair_Draw( void *self )`
- **Purpose:** Owner-draw callback for the crosshair list widget; renders the label, focus highlight cursor, and a 24×24 preview of the selected crosshair shader.
- **Inputs:** `self` — pointer to the `menulist_s` crosshair widget
- **Outputs/Return:** void
- **Side effects:** Issues renderer draw calls via `UI_FillRect`, `UI_DrawChar`, `UI_DrawString`, `UI_DrawHandlePic`
- **Calls:** `UI_FillRect`, `UI_DrawChar`, `UI_DrawString`, `UI_DrawHandlePic`
- **Notes:** Returns early without drawing the shader icon when `curvalue == 0` (no crosshair).

### Preferences_MenuInit
- **Signature:** `static void Preferences_MenuInit( void )`
- **Purpose:** Zeroes `s_preferences`, sets up all widget structs with positions/flags/callbacks, adds them to the menu framework, and calls `Preferences_SetMenuItems` to sync with current cvars.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes all of `s_preferences`; calls `Preferences_Cache` for asset loading; calls `Menu_AddItem` 14 times
- **Calls:** `Preferences_Cache`, `Menu_AddItem`, `Preferences_SetMenuItems`
- **Notes:** Vertical layout is managed via an incrementing `y` variable starting at 144, stepped by `BIGCHAR_HEIGHT + 2` per row.

### Preferences_Cache
- **Signature:** `void Preferences_Cache( void )`
- **Purpose:** Registers all UI art shaders and all 10 crosshair shaders (`gfx/2d/crosshaira` … `gfx/2d/crosshairj`) with the renderer.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Populates `s_preferences.crosshairShader[]`; allocates renderer handles
- **Calls:** `trap_R_RegisterShaderNoMip`, `va`
- **Notes:** Called during menu pre-caching as well as from `Preferences_MenuInit`.

### UI_PreferencesMenu
- **Signature:** `void UI_PreferencesMenu( void )`
- **Purpose:** Public entry point; initializes the menu and pushes it onto the UI stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `Preferences_MenuInit`, then `UI_PushMenu`
- **Calls:** `Preferences_MenuInit`, `UI_PushMenu`

## Control Flow Notes
This file is driven entirely by the UI menu system event loop. `UI_PreferencesMenu` is called once to push the screen; thereafter `Preferences_Event` is invoked per-frame by the menu framework whenever the player activates a widget. There is no per-frame update logic — all state changes are event-driven.

## External Dependencies
- **`ui_local.h`** — menu framework types (`menuframework_s`, `menuradiobutton_s`, etc.), `trap_*` syscall wrappers, `UI_Push/PopMenu`, draw utilities
- **`trap_Cvar_VariableValue` / `trap_Cvar_SetValue` / `trap_Cvar_Reset`** — VM syscall layer (defined in `ui_syscalls.c`)
- **`trap_R_RegisterShaderNoMip`** — renderer syscall (defined in `ui_syscalls.c`)
- **`Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu`** — defined in `ui_atoms.c` / `ui_qmenu.c`
- **`Com_Clamp`** — defined in `game/q_shared.c`
- **cvars touched:** `cg_drawCrosshair`, `cg_simpleItems`, `cg_brassTime`, `cg_marks`, `cg_drawCrosshairNames`, `r_dynamiclight`, `r_fastsky`, `r_finish`, `cg_forcemodel`, `cg_drawTeamOverlay`, `cl_allowDownload`, `sv_allowDownload`
