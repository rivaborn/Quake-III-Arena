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

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `bind_t` | struct | Describes one bindable action: engine command, display label, action ID, preview animation, default/current bind slots |
| `configcvar_t` | struct | Tracks a cvar's name, saved default value, and current value for diff/restore |
| `controls_t` | struct | Monolithic menu state: all `menuXxx_s` widget instances, player preview state, section index, `waitingforkey` flag |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_controls` | `controls_t` | static (file) | Singleton menu instance holding all widget and player-preview state |
| `controls_binding_color` | `vec4_t` | static (file) | Orange color used to draw unselected binding labels |
| `g_bindings[]` | `bind_t[]` | static (file) | Master table of all 34 bindable actions with defaults and live bind slots |
| `g_configcvars[]` | `configcvar_t[]` | static (file) | Table of 7 mouse/joystick/run cvars cached at menu open |
| `g_movement_controls[]` … `g_misc_controls[]` | `menucommon_s*[]` | static (file) | Per-section ordered item lists used for show/hide and layout |
| `g_controls[]` | `menucommon_s**[]` | static (file) | Four-element array indexed by `s_controls.section` |

## Key Functions / Methods

### Controls_InitCvars
- **Signature:** `static void Controls_InitCvars(void)`
- **Purpose:** Snapshot current and default values of all tracked cvars into `g_configcvars[]`.
- **Inputs:** None (reads `g_configcvars[]` table)
- **Outputs/Return:** void; populates `cvarptr->value` and `cvarptr->defaultvalue`
- **Side effects:** Calls `trap_Cvar_Reset` then `trap_Cvar_SetValue` per cvar (temporary engine state change, then restored)
- **Calls:** `trap_Cvar_VariableValue`, `trap_Cvar_Reset`, `trap_Cvar_SetValue`

### Controls_GetConfig
- **Signature:** `static void Controls_GetConfig(void)`
- **Purpose:** Populate `g_bindings[].bind1/bind2` from live engine key table; read cvar widget values.
- **Inputs:** None
- **Outputs/Return:** void; mutates `g_bindings[]` and `s_controls.*` widget `curvalue` fields
- **Side effects:** Calls `trap_Key_GetBindingBuf` for all 256 key slots
- **Calls:** `Controls_GetKeyAssignment`, `Controls_GetCvarValue`, `UI_ClampCvar`

### Controls_SetConfig
- **Signature:** `static void Controls_SetConfig(void)`
- **Purpose:** Flush local binding store and widget values back to the engine.
- **Side effects:** Calls `trap_Key_SetBinding` for every bound key; sets cvars; executes `in_restart`
- **Calls:** `trap_Key_SetBinding`, `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_Cmd_ExecuteText`

### Controls_MenuKey
- **Signature:** `static sfxHandle_t Controls_MenuKey(int key)`
- **Purpose:** Central key handler. When `waitingforkey` is set, captures the next key and assigns it to the focused binding, evicting conflicts. Handles BACKSPACE/DEL to clear.
- **Inputs:** `key` — raw key code from UI layer
- **Outputs/Return:** `sfxHandle_t` menu sound
- **Side effects:** Modifies `g_bindings[]`, calls `trap_Key_SetBinding`, sets `s_controls.waitingforkey`
- **Calls:** `Controls_SetConfig`, `Controls_Update`, `Menu_DefaultKey`
- **Notes:** Uses `goto ignorekey` for early exits; `K_CHAR_FLAG` keys are always ignored while capturing.

### Controls_Update
- **Signature:** `static void Controls_Update(void)`
- **Purpose:** Hide/show/position menu items based on active section; gray all items when `waitingforkey`; highlight the active tab button.
- **Side effects:** Modifies `flags`, `x`, `y`, `left`, `right`, `top`, `bottom` on all widget `menucommon_s` structs
- **Calls:** None (pure state mutation)

### Controls_UpdateModel
- **Signature:** `static void Controls_UpdateModel(int anim)`
- **Purpose:** Map an `ANIM_*` constant to player animation/weapon/pose state and call `UI_PlayerInfo_SetInfo`.
- **Side effects:** Mutates `s_controls.playerLegs/Torso/Weapon/Chat/ViewAngles/MoveAngles`
- **Calls:** `UI_PlayerInfo_SetInfo`

### Controls_DrawKeyBinding
- **Signature:** `static void Controls_DrawKeyBinding(void *self)`
- **Purpose:** Custom ownerdraw for each binding action item. Displays label + bound key name; shows capture prompt when focused.
- **Calls:** `Menu_ItemAtCursor`, `trap_Key_KeynumToStringBuf`, `Q_strupr`, `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`

### Controls_MenuInit
- **Signature:** `static void Controls_MenuInit(void)`
- **Purpose:** Zero and configure all widgets, add them to the menu framework, load config, init model and weapons.
- **Side effects:** Allocates no heap; registers shaders/models via trap calls; populates `s_controls` completely
- **Calls:** `Controls_Cache`, `Controls_InitCvars`, `Controls_GetConfig`, `Controls_InitModel`, `Controls_InitWeapons`, `Controls_Update`, `Menu_AddItem`, `trap_Cvar_VariableStringBuffer`, `Q_CleanStr`

### Controls_Cache / UI_ControlsMenu
- **Notes:** `Controls_Cache` pre-registers the four UI art shaders. `UI_ControlsMenu` is the public entry point: calls `Controls_MenuInit` then `UI_PushMenu`.

## Control Flow Notes
- **Init:** `UI_ControlsMenu` → `Controls_MenuInit` → loads cvars, bindings, model, weapons; sets default section to `C_LOOKING`; calls `Controls_Update`.
- **Frame:** Menu framework calls per-item `ownerdraw` (`Controls_DrawKeyBinding`, `Controls_DrawPlayer`) each render frame.
- **Input:** `Controls_MenuKey` is installed as `s_controls.menu.key`; intercepts all keys before the default handler.
- **Shutdown/save:** On back/exit events, `Controls_SetConfig` is called if `changesmade` is set, then `UI_PopMenu`.

## External Dependencies
- **Includes:** `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h`, `keycodes.h`, `tr_types.h`
- **Defined elsewhere:** `trap_Key_*`, `trap_Cvar_*`, `trap_R_RegisterModel/Shader`, `trap_Cmd_ExecuteText` (syscall stubs in `ui_syscalls.c`); `UI_PlayerInfo_SetModel/SetInfo`, `UI_DrawPlayer` (`ui_players.c`); `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor` (`ui_qmenu.c`); `UI_ConfirmMenu` (`ui_confirm.c`); `bg_itemlist` (`bg_misc.c`)
