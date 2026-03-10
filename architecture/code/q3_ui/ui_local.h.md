# code/q3_ui/ui_local.h

## File Purpose
Central internal header for the legacy `q3_ui` UI module. It declares all shared types, constants, extern variables, and function prototypes used across the UI subsystem's many `.c` source files.

## Core Responsibilities
- Define the menu-item type system (`MTYPE_*`) and flag bitmask (`QMF_*`) constants
- Declare all menu widget structs (`menuframework_s`, `menucommon_s`, `menufield_s`, `menuslider_s`, `menulist_s`, etc.)
- Declare the top-level UI state singleton `uiStatic_t uis`
- Expose `vmCvar_t` extern declarations for all UI-owned cvars
- Declare the full set of `trap_*` syscall wrappers used by UI VM code
- Forward-declare all per-screen cache/init/draw entry points across every UI screen file
- Declare the `playerInfo_t` / `lerpFrame_t` types used for 3D player preview rendering

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `menuframework_s` | struct | Root container for a menu page; holds item list, draw/key callbacks, cursor state |
| `menucommon_s` | struct | Base "vtable-like" header embedded in every menu widget; holds type, position, callbacks |
| `mfield_t` | struct | Raw edit-field state: buffer, cursor, scroll, maxchars |
| `menufield_s` | struct | Widget wrapper combining `menucommon_s` + `mfield_t` |
| `menuslider_s` | struct | Slider widget with min/max/curvalue/range |
| `menulist_s` | struct | Scrollable list widget with item names, columns, separation |
| `menuaction_s` | struct | Simple activatable button (generic only) |
| `menuradiobutton_s` | struct | Two-state toggle widget |
| `menubitmap_s` | struct | Image/shader widget with optional focus shader and color |
| `menutext_s` | struct | Static or styled text label |
| `lerpFrame_t` | struct | Interpolation state for a single skeletal animation part (frame, time, backlerp) |
| `playerInfo_t` | struct | Full model+animation state for the 3D player preview (legs, torso, head, weapon) |
| `uiStatic_t` | struct | Global UI singleton: timing, cursor, menu stack, GL config, shared shader handles |
| `awardType_t` | enum | Award category constants (accuracy, impressive, excellent, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `uis` | `uiStatic_t` | global (extern) | Single UI runtime state instance; owns menu stack, timing, shared resources |
| `m_entersound` | `qboolean` | global (extern) | Flag to play the menu-enter sound on next frame |
| `menu_in_sound` … `weaponChangeSound` | `sfxHandle_t` | global (extern, ×6) | Shared menu audio handles registered at cache time |
| `menu_text_color` … `text_color_highlight` | `vec4_t` | global (extern, ×17) | Shared RGBA color constants for all menu drawing |
| `ui_ffa_fraglimit` … `ui_cdkeychecked` | `vmCvar_t` | global (extern, ×40+) | VM cvar mirrors for all UI-owned console variables |

## Key Functions / Methods

This is a **header-only declaration file**; no function bodies are defined here. All entries below are forward declarations.

### trap_* syscall wrappers (ui_syscalls.c)
- **Signature:** e.g. `void trap_Cvar_Set(const char *var_name, const char *value)`
- **Purpose:** Thin VM-to-engine bridge; each wraps a `syscall()` into the engine import table (`uiImport_t` enum from `ui_public.h`).
- **Inputs:** Vary per call — cvar names, shader paths, render structs, key numbers, LAN indices, etc.
- **Outputs/Return:** Vary — handles, counts, boolean flags, or void.
- **Side effects:** Cross VM boundary; engine-side I/O, renderer state, audio, file system, key system.
- **Notes:** `trap_Cmd_ExecuteText` comment warns never to use `EXEC_NOW` from VM context.

### UI_Init / UI_Shutdown / UI_Refresh / UI_KeyEvent / UI_MouseEvent (ui_atoms.c)
- **Purpose:** Engine-facing export entry points matching `uiExport_t`; drive the entire UI lifecycle and per-frame update.
- **Notes:** `UI_Refresh(int realtime)` is declared twice — likely one declaration is redundant.

### UI_DrawPlayer (ui_players.c)
- **Signature:** `void UI_DrawPlayer(float x, float y, float w, float h, playerInfo_t *pi, int time)`
- **Purpose:** Renders an animated 3D player model preview into a screen rect using `refEntity_t` submission.
- **Inputs:** Screen rect, filled `playerInfo_t`, current time in ms.
- **Side effects:** Calls `trap_R_AddRefEntityToScene`, `trap_R_RenderScene`.

### UI_PlayerInfo_SetModel / UI_PlayerInfo_SetInfo / UI_RegisterClientModelname (ui_players.c)
- **Purpose:** Load and configure a `playerInfo_t` from a model/skin name string; set animation and view state.
- **Notes:** `UI_RegisterClientModelname` returns `qboolean`; failure should fall back to a default model.

### Menu_Draw / Menu_DefaultKey / Menu_AdjustCursor (ui_qmenu.c)
- **Purpose:** Core menu framework rendering and input dispatch; iterate `menuframework_s.items[]` and call per-widget draw/key handlers.

## Control Flow Notes
This header is included by every `q3_ui/*.c` translation unit. At startup `UI_Init` → `UI_RegisterCvars` + `Menu_Cache` initialize shared resources. Each frame `UI_Refresh` drives `Menu_Draw` on `uis.activemenu`. `UI_KeyEvent` / `UI_MouseEvent` feed into `Menu_DefaultKey` / `ScrollList_Key` etc. `UI_Shutdown` tears down. The module is loaded as a QVM or DLL; all engine calls go through the `trap_*` wrappers.

## External Dependencies
- `game/q_shared.h` — core types (`vec3_t`, `qboolean`, `vmCvar_t`, `sfxHandle_t`, etc.)
- `cgame/tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t`)
- `ui/ui_public.h` — `uiExport_t`, `uiImport_t`, `uiMenuCommand_t`, `uiClientState_t` (imported from new UI; `UI_API_VERSION` overridden to 4)
- `keycodes.h` — `keyNum_t` enum, `K_CHAR_FLAG`
- `game/bg_public.h` — `weapon_t`, `animation_t`, `animNumber_t`, `MAX_ANIMATIONS`, game-type enums
- **Defined elsewhere:** All `trap_*` bodies (`ui_syscalls.c`), all `Menu_*` / `Bitmap_*` / `ScrollList_*` bodies (`ui_qmenu.c`), all per-screen `*_Cache` / `UI_*Menu` functions in their respective `.c` files.
