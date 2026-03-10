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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `saveConfig_t` | struct | Aggregates all UI widgets for the Save Config screen into a single state container |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `saveConfig` | `saveConfig_t` (static) | file-static | Sole instance of the Save Config menu state; reinitialized on each menu open |

## Key Functions / Methods

### UI_SaveConfigMenu_BackEvent
- **Signature:** `static void UI_SaveConfigMenu_BackEvent( void *ptr, int event )`
- **Purpose:** Callback for the Back button; dismisses the Save Config menu.
- **Inputs:** `ptr` — pointer to the menu item (unused); `event` — UI callback event code
- **Outputs/Return:** `void`
- **Side effects:** Calls `UI_PopMenu()`, removing the top menu from the navigation stack
- **Calls:** `UI_PopMenu`
- **Notes:** Guards on `event != QM_ACTIVATED`; no-ops for non-activation events

---

### UI_SaveConfigMenu_SaveEvent
- **Signature:** `static void UI_SaveConfigMenu_SaveEvent( void *ptr, int event )`
- **Purpose:** Callback for the Save button; strips the extension from the typed name and issues a `writeconfig` command, then pops the menu.
- **Inputs:** `ptr` — unused; `event` — UI callback event code
- **Outputs/Return:** `void`
- **Side effects:** Appends `writeconfig <name>.cfg\n` to the command buffer via `trap_Cmd_ExecuteText(EXEC_APPEND, ...)`; calls `UI_PopMenu`
- **Calls:** `COM_StripExtension`, `trap_Cmd_ExecuteText`, `va`, `UI_PopMenu`
- **Notes:** Early-outs if `event != QM_ACTIVATED` or if the filename buffer is empty; `COM_StripExtension` prevents double extensions

---

### UI_SaveConfigMenu_SavenameDraw
- **Signature:** `static void UI_SaveConfigMenu_SavenameDraw( void *self )`
- **Purpose:** Owner-draw callback that renders the filename input field with a label, black fill behind the text area, and appropriate highlight/pulse style when focused.
- **Inputs:** `self` — pointer to the `menufield_s` being drawn
- **Outputs/Return:** `void`
- **Side effects:** Issues draw calls: `UI_DrawProportionalString`, `UI_FillRect`, `MField_Draw`
- **Calls:** `Menu_ItemAtCursor`, `UI_DrawProportionalString`, `UI_FillRect`, `MField_Draw`
- **Notes:** Uses `text_color_highlight` + `UI_PULSE` when focused; `colorRed` otherwise

---

### UI_SaveConfigMenu_Init
- **Signature:** `static void UI_SaveConfigMenu_Init( void )`
- **Purpose:** Zeros the `saveConfig` state, sets menu flags, and populates all widget structs with positions, art references, callbacks, and IDs before registering them with `Menu_AddItem`.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `UI_SaveConfigMenu_Cache`; mutates the file-static `saveConfig`; registers 5 items into `saveConfig.menu`
- **Calls:** `memset`, `UI_SaveConfigMenu_Cache`, `Menu_AddItem`
- **Notes:** Layout targets a fixed 640×480 virtual resolution; the filename field is capped at 20 uppercase characters

---

### UI_SaveConfigMenu_Cache
- **Signature:** `void UI_SaveConfigMenu_Cache( void )`
- **Purpose:** Pre-loads all bitmap shaders used by this menu into the renderer's cache.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for 5 art paths
- **Calls:** `trap_R_RegisterShaderNoMip` ×5
- **Notes:** Public; called both from `UI_SaveConfigMenu_Init` and potentially from a global cache-warming pass

---

### UI_SaveConfigMenu
- **Signature:** `void UI_SaveConfigMenu( void )`
- **Purpose:** Public entry point — initializes the menu and pushes it onto the UI navigation stack.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `UI_SaveConfigMenu_Init`, then `UI_PushMenu`
- **Calls:** `UI_SaveConfigMenu_Init`, `UI_PushMenu`
- **Notes:** Every call resets `saveConfig` (via `Init`), clearing any previously typed filename

## Control Flow Notes
This file has no frame/update involvement. It participates purely in **menu navigation**: `UI_SaveConfigMenu` is called once to push the screen, user interaction triggers the two event callbacks, and either callback calls `UI_PopMenu` to return. There is no per-frame tick or render hook; drawing is handled by the `menuframework_s` draw pipeline via the registered owner-draw callback.

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
