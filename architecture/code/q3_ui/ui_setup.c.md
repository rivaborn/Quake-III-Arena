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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `setupMenuInfo_t` | struct | Aggregates the `menuframework_s` and all child menu item widgets for the Setup screen |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `setupMenuInfo` | `setupMenuInfo_t` | static (file) | Sole instance of the Setup menu; owns all widget state for this screen |

## Key Functions / Methods

### Setup_ResetDefaults_Action
- **Signature:** `static void Setup_ResetDefaults_Action( qboolean result )`
- **Purpose:** Callback invoked after the user confirms or cancels the "SET TO DEFAULTS?" dialog.
- **Inputs:** `result` — `qtrue` if user confirmed, `qfalse` if cancelled.
- **Outputs/Return:** void
- **Side effects:** On confirmation, queues three console commands: `exec default.cfg`, `cvar_restart`, `vid_restart` — fully resetting and restarting the renderer.
- **Calls:** `trap_Cmd_ExecuteText`
- **Notes:** Early-out on `result == qfalse`; `vid_restart` causes a full renderer restart.

---

### Setup_ResetDefaults_Draw
- **Signature:** `static void Setup_ResetDefaults_Draw( void )`
- **Purpose:** Custom draw callback for the confirm dialog, rendering a two-line yellow warning message.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Draws to the screen at fixed coordinates relative to `SCREEN_WIDTH/2`.
- **Calls:** `UI_DrawProportionalString`
- **Notes:** Passed as a function pointer to `UI_ConfirmMenu`.

---

### UI_SetupMenu_Event
- **Signature:** `static void UI_SetupMenu_Event( void *ptr, int event )`
- **Purpose:** Unified callback for all interactive menu items; dispatches on item ID to open the appropriate sub-menu.
- **Inputs:** `ptr` — pointer to `menucommon_s` (cast internally); `event` — event type constant.
- **Outputs/Return:** void
- **Side effects:** Pushes new menus onto the UI stack or pops the current menu.
- **Calls:** `UI_PlayerSettingsMenu`, `UI_ControlsMenu`, `UI_GraphicsOptionsMenu`, `UI_PreferencesMenu`, `UI_CDKeyMenu`, `UI_ConfirmMenu`, `UI_PopMenu`
- **Notes:** Ignores all events except `QM_ACTIVATED`. Load/Save cases are permanently commented out.

---

### UI_SetupMenu_Init
- **Signature:** `static void UI_SetupMenu_Init( void )`
- **Purpose:** Allocates and populates all widget fields in `setupMenuInfo`, then registers each with the menu framework.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes to file-static `setupMenuInfo`; calls `Menu_AddItem` for each widget; conditionally omits "DEFAULTS" when `cl_paused != 0`.
- **Calls:** `UI_SetupMenu_Cache`, `memset`, `trap_Cvar_VariableValue`, `Menu_AddItem`
- **Notes:** Load/Save items are `#if 0`-guarded and dead. Vertical spacing is driven by the constant `SETUP_MENU_VERTICAL_SPACING` (34 px), starting at y=134.

---

### UI_SetupMenu_Cache
- **Signature:** `void UI_SetupMenu_Cache( void )`
- **Purpose:** Pre-registers all bitmap shaders needed by the Setup menu so they are GPU-resident before display.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Registers four shader handles via the renderer trap.
- **Calls:** `trap_R_RegisterShaderNoMip`

---

### UI_SetupMenu
- **Signature:** `void UI_SetupMenu( void )`
- **Purpose:** Public entry point — initializes and pushes the Setup menu onto the active UI stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Calls `UI_SetupMenu_Init`, then `UI_PushMenu`.
- **Calls:** `UI_SetupMenu_Init`, `UI_PushMenu`

## Control Flow Notes
This file participates in the **UI menu stack** only. `UI_SetupMenu` is called by a parent menu (e.g., the main menu or in-game menu). There is no per-frame update or render loop here; rendering is handled generically by `Menu_Draw` via `menuframework_s.draw`. The `cl_paused` cvar check at init time gates the "DEFAULTS" item, preventing config reset from an active game session.

## External Dependencies
- **Includes:** `ui_local.h` (aggregates `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework declarations, and all trap syscall prototypes)
- **Defined elsewhere:**
  - `UI_PlayerSettingsMenu`, `UI_ControlsMenu`, `UI_GraphicsOptionsMenu`, `UI_PreferencesMenu`, `UI_CDKeyMenu` — sub-menu entry points in their respective `.c` files
  - `UI_ConfirmMenu` — `ui_confirm.c`
  - `UI_PushMenu`, `UI_PopMenu` — `ui_atoms.c`
  - `Menu_AddItem`, `Menu_Draw` — `ui_qmenu.c`
  - `trap_*` syscalls — `ui_syscalls.c` (VM trap layer)
  - `color_white`, `color_red`, `color_yellow` — `ui_atoms.c`
