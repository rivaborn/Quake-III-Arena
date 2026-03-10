# code/q3_ui/ui_options.c

## File Purpose
Implements the top-level "System Setup" options menu for Quake III Arena's legacy UI module. It presents four sub-menu navigation buttons (Graphics, Display, Sound, Network) plus a Back button, acting as a hub that dispatches to each specialized settings screen.

## Core Responsibilities
- Initialize and layout the System Setup menu (`optionsmenu_t`) with all UI items
- Pre-cache all artwork (frame bitmaps, back button) used by this menu
- Route activation events to the appropriate sub-menu or pop the menu stack
- Conditionally set fullscreen mode based on whether the client is already connected

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `optionsmenu_t` | struct (typedef) | Aggregates all menu items for the System Setup screen: banner, decorative frames, four nav buttons, and a back button |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_options` | `optionsmenu_t` | static (file) | Persistent storage for the single System Setup menu instance |

## Key Functions / Methods

### Options_Event
- **Signature:** `static void Options_Event( void* ptr, int event )`
- **Purpose:** Callback invoked by the menu system when a menu item is activated. Dispatches to the appropriate sub-menu or pops the stack.
- **Inputs:** `ptr` — pointer to the triggering `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** `void`
- **Side effects:** Pushes a new menu via `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`, or pops the current menu via `UI_PopMenu`
- **Calls:** `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`, `UI_PopMenu`
- **Notes:** Early-returns silently for any event that is not `QM_ACTIVATED`; no fallthrough for unrecognized IDs

### SystemConfig_Cache
- **Signature:** `void SystemConfig_Cache( void )`
- **Purpose:** Pre-registers all artwork shaders used by this menu so they are resident in the renderer cache before drawing begins.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for `ART_FRAMEL`, `ART_FRAMER`, `ART_BACK0`, `ART_BACK1`
- **Calls:** `trap_R_RegisterShaderNoMip` (×4)
- **Notes:** Intended to be called during load/precache phases, not per-frame

### Options_MenuInit
- **Signature:** `void Options_MenuInit( void )`
- **Purpose:** Zeroes, configures, and populates `s_options` with all widget properties, then adds every item to the menu framework.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Mutates global `s_options`; calls `trap_GetClientState` to determine fullscreen mode; calls `SystemConfig_Cache`; calls `Menu_AddItem` for each of eight widgets
- **Calls:** `memset`, `SystemConfig_Cache`, `trap_GetClientState`, `Menu_AddItem` (×8)
- **Notes:** `y` is incremented by `VERTICAL_SPACING` (34px) between each nav button, starting at y=168. Back button is pinned to y=416 (480−64) regardless of spacing.

### UI_SystemConfigMenu
- **Signature:** `void UI_SystemConfigMenu( void )`
- **Purpose:** Public entry point — initializes the menu and pushes it onto the UI menu stack, making it the active menu.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `Options_MenuInit` (reinitializes `s_options`), then `UI_PushMenu`
- **Calls:** `Options_MenuInit`, `UI_PushMenu`
- **Notes:** This is the only externally-visible function in this file

## Control Flow Notes
This file has no frame/update loop participation. It is invoked on-demand: `UI_SystemConfigMenu` is called from another menu (e.g., the main menu or in-game menu), which pushes this screen. All subsequent lifecycle (draw, key handling) is managed by the generic `menuframework_s` machinery in `ui_qmenu.c`. Navigation away from this screen either pushes a sub-menu (leaving this on the stack) or calls `UI_PopMenu` to return to the caller.

## External Dependencies
- **Includes:** `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- **Defined elsewhere:**
  - `Menu_AddItem`, `UI_PushMenu`, `UI_PopMenu` — `ui_qmenu.c` / `ui_atoms.c`
  - `UI_GraphicsOptionsMenu`, `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu` — respective `ui_video.c`, `ui_display.c`, `ui_sound.c`, `ui_network.c`
  - `trap_R_RegisterShaderNoMip`, `trap_GetClientState` — `ui_syscalls.c` (VM syscall wrappers)
  - `color_red`, `color_white` — `ui_atoms.c`
