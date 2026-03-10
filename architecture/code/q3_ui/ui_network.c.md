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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `networkOptionsInfo_t` | struct | Aggregates all menu widgets (banner, frame bitmaps, tab labels, rate spin-control, back button) into a single layout descriptor |
| `menuframework_s` | struct (defined elsewhere) | Core menu container holding item list, cursor state, and draw/key callbacks |
| `menutext_s` | struct (defined elsewhere) | Proportional or banner text widget |
| `menubitmap_s` | struct (defined elsewhere) | Bitmap/shader widget with optional focus image |
| `menulist_s` | struct (defined elsewhere) | Spin-control widget backed by a `const char **` item array |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `networkOptionsInfo` | `networkOptionsInfo_t` | static (file) | Single persistent instance of the network options menu layout; zeroed on each `_Init` call |
| `rate_items` | `const char *[]` | static (file) | Null-terminated string array of human-readable rate tier labels shown in the spin-control |

## Key Functions / Methods

### UI_NetworkOptionsMenu_Event
- **Signature:** `static void UI_NetworkOptionsMenu_Event( void* ptr, int event )`
- **Purpose:** Unified callback for all interactive widgets; dispatches on widget ID.
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — notification type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** void
- **Side effects:** Pops the current menu and pushes a sibling options menu (`ID_GRAPHICS/DISPLAY/SOUND`); writes `rate` cvar via `trap_Cvar_SetValue` (`ID_RATE`); pops menu on `ID_BACK`
- **Calls:** `UI_PopMenu`, `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `trap_Cvar_SetValue`
- **Notes:** `ID_NETWORK` case is a no-op (current screen is already Network). Rate values are hard-coded: 2500 / 3000 / 4000 / 5000 / 25000.

### UI_NetworkOptionsMenu_Init
- **Signature:** `static void UI_NetworkOptionsMenu_Init( void )`
- **Purpose:** Zeroes the state struct, configures every widget's position/style/callback, adds all items to the menu framework, and reads the current `rate` cvar to set the spin-control's initial selection.
- **Inputs:** none (reads `rate` cvar via `trap_Cvar_VariableValue`)
- **Outputs/Return:** void
- **Side effects:** Mutates `networkOptionsInfo` global; calls `UI_NetworkOptionsMenu_Cache`; reads `rate` cvar
- **Calls:** `memset`, `UI_NetworkOptionsMenu_Cache`, `Menu_AddItem`, `trap_Cvar_VariableValue`
- **Notes:** Rate-to-index mapping uses `<=` thresholds mirroring the inverse write-back in `_Event`. Layout is fixed at 640×480 virtual resolution.

### UI_NetworkOptionsMenu_Cache
- **Signature:** `void UI_NetworkOptionsMenu_Cache( void )`
- **Purpose:** Pre-loads all shader/bitmap assets required by this menu so they are resident before drawing begins.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Registers four shaders with the renderer via `trap_R_RegisterShaderNoMip`
- **Calls:** `trap_R_RegisterShaderNoMip` ×4
- **Notes:** Called from `_Init` but also declared externally for pre-caching by the setup menu.

### UI_NetworkOptionsMenu
- **Signature:** `void UI_NetworkOptionsMenu( void )`
- **Purpose:** Public entry point; initializes menu state, pushes it onto the UI stack, and positions the cursor on the "NETWORK" tab label.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Mutates `networkOptionsInfo`; modifies the global UI menu stack
- **Calls:** `UI_NetworkOptionsMenu_Init`, `UI_PushMenu`, `Menu_SetCursorToItem`

## Control Flow Notes
This file has no frame/update loop involvement. `UI_NetworkOptionsMenu` is called once to open the screen. From that point the menu framework drives rendering and input via the registered callback (`UI_NetworkOptionsMenu_Event`). The screen is dismissed by `UI_PopMenu`, either on Back or when switching to a sibling tab.

## External Dependencies
- `ui_local.h` — pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, menu framework types, and all `trap_*` syscall declarations
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `Menu_AddItem`, `Menu_SetCursorToItem`, `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip`, `color_white`, `color_red`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT`
