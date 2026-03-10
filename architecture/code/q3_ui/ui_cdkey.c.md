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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `cdkeyMenuInfo_t` | struct | Aggregates all menu widgets: framework, banner text, decorative frame bitmap, key input field, accept and back buttons |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cdkeyMenuInfo` | `cdkeyMenuInfo_t` | static (file) | Single instance of the CD Key menu; re-initialized each time the menu is opened |

## Key Functions / Methods

### UI_CDKeyMenu_Event
- **Signature:** `static void UI_CDKeyMenu_Event( void *ptr, int event )`
- **Purpose:** Callback for Accept/Back button activation events
- **Inputs:** `ptr` — pointer to `menucommon_s` with `.id`; `event` — menu event type
- **Outputs/Return:** void
- **Side effects:** On `ID_ACCEPT`, calls `trap_SetCDKey` if the buffer is non-empty; calls `UI_PopMenu` in both cases
- **Calls:** `trap_SetCDKey`, `UI_PopMenu`
- **Notes:** Filters on `QM_ACTIVATED`; no-ops on all other events

### UI_CDKeyMenu_PreValidateKey
- **Signature:** `static int UI_CDKeyMenu_PreValidateKey( const char *key )`
- **Purpose:** Client-side format check — length must be 16, and every character must belong to the allowed set (`2 3 7 a b c d g h j l p r s t w`)
- **Inputs:** `key` — null-terminated key string from the input buffer
- **Outputs/Return:** `1` = incomplete (length ≠ 16), `0` = appears valid, `-1` = invalid character found
- **Side effects:** None
- **Calls:** `strlen`
- **Notes:** Does not perform cryptographic verification; that is delegated to `trap_VerifyCDKey`

### UI_CDKeyMenu_DrawKey
- **Signature:** `static void UI_CDKeyMenu_DrawKey( void *self )`
- **Purpose:** Owner-draw callback for the CD key `menufield_s`; renders the input buffer, cursor, and a status string based on pre-validation result
- **Inputs:** `self` — cast to `menufield_s *`
- **Outputs/Return:** void
- **Side effects:** Issues renderer draw calls via `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`, `UI_DrawProportionalString`
- **Calls:** `trap_Key_GetOverstrikeMode`, `UI_CDKeyMenu_PreValidateKey`, `UI_FillRect`, `UI_DrawString`, `UI_DrawChar`, `UI_DrawProportionalString`
- **Notes:** Status messages are color-coded: yellow = incomplete, white = valid, red = invalid

### UI_CDKeyMenu_Init
- **Signature:** `static void UI_CDKeyMenu_Init( void )`
- **Purpose:** Sets `ui_cdkeychecked` cvar, zeroes and populates `cdkeyMenuInfo` with widget parameters, conditionally adds the Back button (only when a previous menu exists on the stack via `uis.menusp`), retrieves the stored CD key, and clears it if `trap_VerifyCDKey` fails
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies `cdkeyMenuInfo` (static global); sets cvar `ui_cdkeychecked`; calls `trap_GetCDKey` and optionally `trap_VerifyCDKey`
- **Calls:** `trap_Cvar_Set`, `UI_CDKeyMenu_Cache`, `memset`, `Menu_AddItem`, `trap_GetCDKey`, `trap_VerifyCDKey`
- **Notes:** Back button is omitted when the CD Key menu is the root/only menu (`uis.menusp == 0`)

### UI_CDKeyMenu_Cache
- **Signature:** `void UI_CDKeyMenu_Cache( void )`
- **Purpose:** Pre-loads all bitmap shaders used by the menu into the renderer cache
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Registers five shaders via `trap_R_RegisterShaderNoMip`
- **Calls:** `trap_R_RegisterShaderNoMip` ×5

### UI_CDKeyMenu / UI_CDKeyMenu_f
- **Notes:** Both are thin public wrappers; `UI_CDKeyMenu` calls `UI_CDKeyMenu_Init` then `UI_PushMenu`. `UI_CDKeyMenu_f` simply delegates to `UI_CDKeyMenu`. These are the external entry points listed in `ui_local.h`.

## Control Flow Notes
This file is entirely UI-frame driven. `UI_CDKeyMenu` / `UI_CDKeyMenu_f` are called once to push the menu onto the stack. Thereafter, each UI refresh tick invokes `Menu_Draw`, which calls the `ownerdraw` hook (`UI_CDKeyMenu_DrawKey`) for the field widget. Input events route through `Menu_DefaultKey` → `UI_CDKeyMenu_Event` on activation. There is no per-frame update logic beyond owner-draw rendering.

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
