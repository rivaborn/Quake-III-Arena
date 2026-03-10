# code/q3_ui/ui_login.c

## File Purpose
Implements the in-game login menu screen for Quake III Arena's online rankings system (GRank). It presents a modal dialog with name and password fields, wiring up input to the rankings authentication syscall.

## Core Responsibilities
- Define and initialize all UI widgets for the login form (frame, labels, text fields, buttons)
- Handle `LOGIN` and `CANCEL` button events via `Login_MenuEvent`
- Submit credentials to the rankings backend via `trap_CL_UI_RankUserLogin`
- Preload/cache the frame shader asset via `Login_Cache`
- Push the menu onto the UI stack via `UI_LoginMenu`

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `login_t` | struct | Aggregates all menu widgets for the login screen into one layout structure |
| `menuframework_s` | struct (defined in `ui_local.h`) | Base menu container, holds item list and navigation state |
| `menubitmap_s` | struct (defined in `ui_local.h`) | Decorative frame bitmap widget |
| `menufield_s` | struct (defined in `ui_local.h`) | Editable text input field with a buffer |
| `menutext_s` | struct (defined in `ui_local.h`) | Clickable or static proportional text label |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_login` | `login_t` | static | Single instance of the login menu widget tree |
| `s_login_menu` | `menuframework_s` | static | Declared but unused; `s_login.menu` is used instead |
| `s_login_login` | `menuaction_s` | static | Declared but unused; superseded by `menutext_s` items in `login_t` |
| `s_login_cancel` | `menuaction_s` | static | Declared but unused; superseded by `menutext_s` items in `login_t` |
| `s_login_color_prompt` | `vec4_t` | static | Orange `{1.00, 0.43, 0.00, 1.00}` used for label text color |

## Key Functions / Methods

### Login_MenuEvent
- **Signature:** `static void Login_MenuEvent( void* ptr, int event )`
- **Purpose:** Callback for LOGIN and CANCEL button activation.
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is handled)
- **Outputs/Return:** `void`
- **Side effects:** On LOGIN: calls `trap_CL_UI_RankUserLogin` with name/password buffers, then `UI_ForceMenuOff`. On CANCEL: calls `UI_PopMenu`.
- **Calls:** `trap_CL_UI_RankUserLogin`, `UI_ForceMenuOff`, `UI_PopMenu`
- **Notes:** The `trap_Cvar_Set` calls for `name`, `rank_name`, and `rank_pwd` are commented out, suggesting credential CVars were considered but not shipped. Guard on `QM_ACTIVATED` means focus/unfocus events are silently ignored.

### Login_MenuInit
- **Signature:** `void Login_MenuInit( void )`
- **Purpose:** Zero-initialises `s_login`, sets positional and style properties on every widget, then registers all items with the menu framework.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `Login_Cache` for shader preloading; mutates the global `s_login` struct; calls `Menu_AddItem` 7 times.
- **Calls:** `memset`, `Login_Cache`, `Menu_AddItem`
- **Notes:** `y` is a local layout counter incremented by 20 or 40 px between rows. `s_login_menu`, `s_login_login`, `s_login_cancel` are declared at file scope but never populated here — dead state.

### Login_Cache
- **Signature:** `void Login_Cache( void )`
- **Purpose:** Preloads the decorative frame shader into the renderer cache.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for `"menu/art/cut_frame"`.
- **Calls:** `trap_R_RegisterShaderNoMip`

### UI_LoginMenu
- **Signature:** `void UI_LoginMenu( void )`
- **Purpose:** Public entry point — initialises and pushes the login menu onto the UI stack.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `Login_MenuInit`, then `UI_PushMenu`.
- **Calls:** `Login_MenuInit`, `UI_PushMenu`

## Control Flow Notes
`UI_LoginMenu` is the engine-facing entry point, called from rankings/UI dispatch code when a login prompt is needed. It follows the standard Q3 UI pattern: init → push menu. The menu lives on the `uis.stack` until `UI_PopMenu` or `UI_ForceMenuOff` removes it. No per-frame draw or key overrides are registered; the menu relies entirely on the generic `Menu_Draw` / `Menu_DefaultKey` infrastructure.

## External Dependencies
- **`ui_local.h`** — pulls in all menu types, trap wrappers, and helper declarations
- `trap_CL_UI_RankUserLogin` — defined in `ui_syscalls.c`/engine; submits credentials to the rankings server (not declared in the bundled header, implying it is a raw syscall wrapper unique to the GRank module)
- `trap_R_RegisterShaderNoMip` — renderer syscall
- `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` — defined in `ui_atoms.c`
- `Menu_AddItem` — defined in `ui_qmenu.c`
- `Rankings_DrawName`, `Rankings_DrawPassword` — owner-draw callbacks defined in `ui_rankings.c`
