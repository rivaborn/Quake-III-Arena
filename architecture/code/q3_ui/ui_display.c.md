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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `displayOptionsInfo_t` | struct (typedef) | Aggregates all menu widgets for the display options screen into one flat struct |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `displayOptionsInfo` | `displayOptionsInfo_t` | static (file) | Sole persistent instance of the display options menu state; zeroed and rebuilt each time the menu is opened |

## Key Functions / Methods

### UI_DisplayOptionsMenu_Event
- **Signature:** `static void UI_DisplayOptionsMenu_Event( void *ptr, int event )`
- **Purpose:** Unified callback for all interactive widgets; dispatches on widget ID.
- **Inputs:** `ptr` — pointer to the triggering `menucommon_s`; `event` — menu event code (only `QM_ACTIVATED` is acted upon).
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_Cvar_SetValue` to write `r_gamma` or `cg_viewsize`; calls `UI_PopMenu` to dismiss the current screen before pushing a sibling screen.
- **Calls:** `UI_PopMenu`, `UI_GraphicsOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`, `trap_Cvar_SetValue`
- **Notes:** `ID_DISPLAY` case is a no-op (already on this screen). Slider values are scaled: brightness `÷ 10.0f` → `r_gamma`; screen size `× 10` → `cg_viewsize`.

### UI_DisplayOptionsMenu_Init
- **Signature:** `static void UI_DisplayOptionsMenu_Init( void )`
- **Purpose:** Zeroes `displayOptionsInfo`, positions every widget, and adds all items to the menu framework.
- **Inputs:** None (reads `uis.glconfig.deviceSupportsGamma` and cvar values from the engine).
- **Outputs/Return:** `void`
- **Side effects:** Mutates the file-static `displayOptionsInfo`; queries `trap_Cvar_VariableValue` for `r_gamma` and `cg_viewsize` to seed slider positions; calls `UI_DisplayOptionsMenu_Cache`.
- **Calls:** `memset`, `UI_DisplayOptionsMenu_Cache`, `Menu_AddItem`, `trap_Cvar_VariableValue`
- **Notes:** Brightness range is [5, 20] (maps to r_gamma 0.5–2.0); screen size range is [3, 10] (maps to cg_viewsize 30–100). `wrapAround` and `fullscreen` are both set true.

### UI_DisplayOptionsMenu_Cache
- **Signature:** `void UI_DisplayOptionsMenu_Cache( void )`
- **Purpose:** Registers the four UI art shaders with the renderer so they are available when the menu draws.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for `ART_FRAMEL`, `ART_FRAMER`, `ART_BACK0`, `ART_BACK1`.
- **Calls:** `trap_R_RegisterShaderNoMip` (×4)
- **Notes:** Called both from `UI_DisplayOptionsMenu_Init` and potentially pre-emptively by the main cache pass.

### UI_DisplayOptionsMenu
- **Signature:** `void UI_DisplayOptionsMenu( void )`
- **Purpose:** Public entry point; initializes the menu, pushes it onto the UI stack, and places the cursor on the "DISPLAY" tab.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Modifies the global menu stack via `UI_PushMenu`; resets all widget state via `UI_DisplayOptionsMenu_Init`.
- **Calls:** `UI_DisplayOptionsMenu_Init`, `UI_PushMenu`, `Menu_SetCursorToItem`
- **Notes:** Cursor is explicitly forced to `displayOptionsInfo.display` on open to visually indicate the active tab.

## Control Flow Notes
This file has no frame/update loop participation. It is activated on-demand: `UI_DisplayOptionsMenu` is called from a parent menu, pushes the screen onto the stack, and the generic `Menu_Draw` / `Menu_DefaultKey` loop in `ui_qmenu.c` drives rendering and input. On back/navigation events, `UI_PopMenu` removes this screen from the stack.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`)
- **Defined elsewhere:**
  - `uis` (`uiStatic_t`) — global UI state, provides `glconfig.deviceSupportsGamma`
  - `Menu_AddItem`, `Menu_SetCursorToItem`, `UI_PushMenu`, `UI_PopMenu` — menu framework (`ui_qmenu.c` / `ui_atoms.c`)
  - `UI_GraphicsOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu` — sibling screen entry points
  - `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip` — VM syscall wrappers (`ui_syscalls.c`)
  - `color_red`, `color_white`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT` — shared UI constants
