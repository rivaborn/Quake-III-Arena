# code/ui/ui_local.h

## File Purpose
This is the primary internal header for the Quake III Arena UI VM module. It aggregates all type definitions, constants, extern declarations, and trap (syscall) function prototypes needed by the UI subsystem's implementation files.

## Core Responsibilities
- Declares all `vmCvar_t` globals used across UI screens (game rules, server browser, scores, etc.)
- Defines the legacy `menuframework_s` / `menucommon_s` widget type system and associated flags
- Declares the `uiStatic_t` singleton holding frame-level UI state and asset handles
- Declares the large `uiInfo_t` aggregate holding all new-UI runtime state (server lists, maps, tiers, players, mods, demos)
- Provides the complete `trap_*` syscall interface the UI VM uses to call into the engine
- Forward-declares every UI screen module's public cache/init/display functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `menuframework_s` | struct | Container for a legacy menu screen; holds item array, cursor, and draw/key callbacks |
| `menucommon_s` | struct | Base "class" for all legacy menu widgets; position, flags, parent, callbacks |
| `mfield_t` | struct | Text edit field buffer with cursor and scroll state |
| `menufield_s` | struct | Widget wrapping `mfield_t` with `menucommon_s` header |
| `menuslider_s` | struct | Slider widget with min/max/cur value and range |
| `menulist_s` | struct | Scrollable list widget with item name array and multi-column support |
| `menubitmap_s` | struct | Bitmap/image widget with focus shader and color |
| `menutext_s` | struct | Static text label widget with style and color |
| `menuaction_s` | struct | Clickable action button widget |
| `menuradiobutton_s` | struct | Radio button widget with current value |
| `lerpFrame_t` | struct | Interpolated model animation frame state (copied from cgame) |
| `playerInfo_t` | struct | Full player model rendering state for the UI player preview |
| `uiStatic_t` | struct | Per-frame UI globals: time, cursor position, GL config, common shader/font handles |
| `uiInfo_t` | struct | Master new-UI state: server browser, map list, tier list, character list, post-game info, etc. |
| `characterInfo` | struct | Head/character entry (name, image, base model, active flag) |
| `teamInfo` | struct | Team entry with icon handles and member list |
| `mapInfo` | struct | Map entry with load name, image, opponent, game type bits, and per-gametype times |
| `tierInfo` | struct | SP tier entry mapping three maps to game types |
| `serverStatus_t` | struct | Server browser runtime state: ping lists, display server index, MOTD |
| `serverStatusInfo_t` | struct | Cached detailed status for a single queried server |
| `pendingServerStatus_t` | struct | Batch of in-flight server status requests for player-find feature |
| `postGameInfo_t` | struct | Post-match score summary data (score, accuracy, bonuses) |
| `awardType_t` | enum | Award category identifiers used by gameinfo queries |
| `modInfo_t` | struct | Mod name/description entry for the mods menu |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `uis` | `uiStatic_t` | global (extern) | Singleton holding per-frame UI state and cached asset handles |
| `uiInfo` | `uiInfo_t` | global (extern) | Singleton holding all new-UI runtime data |
| `menu_in_sound` … `weaponChangeSound` | `sfxHandle_t` | global (extern) | Shared menu sound effect handles |
| `menu_text_color` … `text_color_highlight` | `vec4_t` | global (extern) | Shared color constants for menu rendering |
| `m_entersound` | `qboolean` | global (extern) | Flag to play the menu-enter sound on next frame |
| `ui_ffa_fraglimit` … `ui_serverStatusTimeOut` | `vmCvar_t` | global (extern) | All registered UI console variables |

## Key Functions / Methods

### trap_* (syscall wrappers)
- Signature: Various — e.g. `void trap_Cvar_Set(const char *var_name, const char *value)`
- Purpose: Thin wrappers that invoke the engine via the VM syscall table. Cover rendering, sound, filesystem, cvars, key system, LAN/server browser, cinematics, and CD key verification.
- Inputs: Depend on specific call.
- Outputs/Return: Depend on specific call.
- Side effects: All engine-side effects (I/O, rendering, network, etc.).
- Calls: Engine syscall dispatch (not visible in this file).
- Notes: The UI VM must never use OS APIs directly; all engine access goes through these traps.

### Menu_* / widget functions (declared extern, implemented in ui_qmenu.c)
- `Menu_Draw`, `Menu_DefaultKey`, `Menu_AdjustCursor`, `Menu_ActivateItem`, `Menu_Focus`, `Menu_SetCursor` — drive the legacy menu widget system each frame.
- `ScrollList_Draw` / `ScrollList_Key` — specialized draw/input for scroll list widgets.
- `Bitmap_Draw` / `Bitmap_Init` — bitmap widget lifecycle.

### MField_* / MenuField_* (ui_mfield.c)
- Handle text input: `MField_KeyDownEvent`, `MField_CharEvent`, `MField_Draw`, `MenuField_Key`.

### UI entry points (ui_main.c / ui_atoms.c)
- `UI_Init`, `UI_Shutdown`, `UI_Refresh`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_ConsoleCommand`, `UI_SetActiveMenu`, `UI_IsFullscreen`, `UI_DrawConnectScreen` — the exported VM entry points called by the engine.

### UI_DrawPlayer / UI_PlayerInfo_* (ui_players.c)
- Signature: `void UI_DrawPlayer(float x, float y, float w, float h, playerInfo_t *pi, int time)`
- Purpose: Renders an animated 3-part player model (legs/torso/head) into a screen rectangle for character selection screens.
- Inputs: Screen rect, populated `playerInfo_t`, current time.
- Outputs/Return: None (render side effect).
- Side effects: Submits ref entities to the renderer via `trap_R_AddRefEntityToScene`.

## Control Flow Notes
This header is included by every `.c` file in `code/ui/`. The module lifecycle is:
1. **Init** — `UI_Init` registers cvars, loads assets, builds menu/map/bot data.
2. **Frame** — `UI_Refresh` is called each client frame; it dispatches to the active menu's draw function.
3. **Input** — `UI_KeyEvent` / `UI_MouseEvent` route input to the active menu's key callback.
4. **Shutdown** — `UI_Shutdown` tears down any persistent state.

## External Dependencies
- `../game/q_shared.h` — base types (`vec3_t`, `qboolean`, `vmCvar_t`, etc.)
- `../cgame/tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, etc.)
- `ui_public.h` — exported UI entry point enum (`uiExport_t`), `uiMenuCommand_t`, `uiClientState_t`
- `keycodes.h` — `keyNum_t` enum
- `../game/bg_public.h` — `weapon_t`, `animation_t`, `animNumber_t`, game type enums
- `ui_shared.h` — new-UI `displayContextDef_t` and shared menu-def types (defined elsewhere)
- Engine syscall dispatch — all `trap_*` targets are defined in the engine, not this module
