# Subsystem Overview

## Purpose
`code/q3_ui` is the legacy client-side UI module for Quake III Arena, compiled as a QVM bytecode shared library and loaded by the engine at runtime. It implements all menu screens, widget rendering, input dispatch, and cvar management for the game's complete interface — from the main menu and single-player progression to multiplayer server browsing, options configuration, and the GRank online rankings overlay.

## Key Files

| File | Role |
|---|---|
| `ui_main.c` | QVM entry point; exports `vmMain`, routes all engine UI commands, owns the cvar registration table |
| `ui_atoms.c` | Core drawing primitives, menu stack management (`UI_PushMenu`/`UI_PopMenu`), per-frame refresh, input dispatch |
| `ui_qmenu.c` | Widget framework: initializes, draws, and routes input for all standard widget types; owns shared asset cache |
| `ui_local.h` | Central internal header; all shared types, widget structs, constants, `uiStatic_t uis` extern, trap prototypes |
| `ui_syscalls.c` | VM syscall wrappers bridging all `trap_*` calls to the engine (not documented individually but referenced throughout) |
| `ui_gameinfo.c` | Arena/bot metadata loading, SP progression tracking, tier video unlock state |
| `ui_players.c` | Animated 3D player model preview: model loading, animation state machines, renderer submission |
| `ui_mfield.c` | Low-level editable text field widget (scrolling, cursor, overstrike, clipboard) |
| `ui_menu.c` | Main menu screen; 3D banner model rendering, top-level navigation hub |
| `ui_ingame.c` | In-game pause menu; conditional item grayout based on runtime cvars |
| `ui_servers2.c` | Multiplayer server browser; ping querying, filtering, sorting, favorites persistence |
| `ui_startserver.c` | Wizard-style server launch flow: map selection → options/bot configuration → `map` command |
| `ui_splevel.c` | Single-player tier/level selection menu with completion status display |
| `ui_sppostgame.c` | Post-match presentation: podium animation, award medals, tier progression, score persistence |
| `ui_controls2.c` | Full keybinding configuration menu with live 3D player preview and tabbed sections |
| `ui_video.c` | Graphics Options and Driver Info menus; renderer cvar writes and `vid_restart` |
| `keycodes.h` | `keyNum_t` enumeration: shared vocabulary for all key event dispatch in the UI |
| `ui_rankings.c` | GRank online rankings overlay menu; context-sensitive options based on rank status |
| `ui_confirm.c` | Reusable modal yes/no confirmation dialog and message box |

## Core Responsibilities

- Expose `vmMain` as the sole engine-facing entry point, routing all UI commands (init, shutdown, key events, mouse events, frame refresh, menu activation, console commands) to internal handlers
- Maintain a menu stack (`UI_PushMenu`/`UI_PopMenu`/`UI_ForceMenuOff`) and dispatch input to the topmost active menu each frame
- Render all 2D UI elements in a virtual 640×480 coordinate space using bitmap font strings, proportional strings, filled rectangles, and named shader handles via `trap_R_*` syscalls
- Manage and synchronize a table of `vmCvar_t` globals mirroring engine cvars, batch-registered via `UI_RegisterCvars` / `UI_UpdateCvars`
- Provide a complete widget framework (buttons, sliders, spin controls, list boxes, text fields, bitmaps, radio buttons) with unified focus, cursor, and sound feedback
- Load, parse, and index arena and bot metadata from `.arena`/`.bot` files to support SP progression, server browser filtering, and bot management
- Drive animated 3D player model previews (multi-part skeletal models, animation state machines, weapon attachment) rendered live within menu screens
- Implement the full single-player campaign flow: level selection, difficulty picking, arena launch, postgame scoring, award tracking, and tier cinematic gating
- Interface with the GRank online rankings backend via GRank-specific `trap_CL_UI_Rank*` syscalls for login, signup, logout, league selection, and status display

## Key Interfaces & Data Flow

**Exposed to the engine:**
- `vmMain(int command, ...)` — the single QVM export; handles `UI_INIT`, `UI_SHUTDOWN`, `UI_KEY_EVENT`, `UI_MOUSE_EVENT`, `UI_REFRESH`, `UI_IS_FULLSCREEN`, `UI_SET_ACTIVE_MENU`, `UI_CONSOLE_COMMAND`, `UI_DRAW_CONNECT_SCREEN`
- `UI_API_VERSION` = 4 (declared in `ui_local.h`, overrides the value from `ui_public.h`)

**Consumed from the engine (via `trap_*` syscall wrappers in `ui_syscalls.c`):**
- **Renderer:** `trap_R_RegisterShaderNoMip`, `trap_R_SetColor`, `trap_R_DrawStretchPic`, `trap_R_RenderScene`, `trap_R_RegisterModel`, `trap_R_RegisterSkin`, `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`
- **Audio:** `trap_S_RegisterSound`, `trap_S_StartLocalSound`
- **Input/Key:** `trap_Key_GetOverstrikeMode`, `trap_Key_SetOverstrikeMode`, `trap_Key_IsDown`, `trap_Key_GetBinding`, `trap_Key_SetBinding`, `trap_Key_ClearStates`, `trap_GetClipboardData`
- **CVar:** `trap_Cvar_Register`, `trap_Cvar_Update`, `trap_Cvar_Set`, `trap_Cvar_SetValue`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cvar_Reset`
- **Command:** `trap_Cmd_ExecuteText`
- **Filesystem:** `trap_FS_GetFileList`
- **Engine state:** `trap_GetClientState`, `trap_GetConfigString`, `trap_LAN_*` (server list/ping), `trap_GetGlconfig`
- **CD key / rankings:** `trap_GetCDKey`, `trap_SetCDKey`, `trap_VerifyCDKey`, `trap_CL_UI_RankUserLogin`, `trap_CL_UI_RankUserCreate`, `trap_CL_UI_RankUserRequestLogout`, `trap_CL_UI_RankGetLeauges`, `trap_CL_UI_RankUserReset`

**Shared types pulled from other subsystems:**
- `game/q_shared.h` — core types (`vec3_t`, `qboolean`, `vmCvar_t`, string utilities)
- `game/bg_public.h` — `weapon_t`, `animation_t`, game-type enums, `bg_itemlist`
- `cgame/tr_types.h` — `refEntity_t`, `refdef_t`, `glconfig_t`
- `ui/ui_public.h` — `uiExport_t`, `uiImport_t`, `uiMenuCommand_t`, `uiClientState_t`

## Runtime Role

**Init (`UI_INIT`):**
`vmMain` routes to `UI_Init` (`ui_atoms.c`), which registers all cvars, loads GL config, calls `Menu_Cache` to preload shared widget assets (shaders, sounds), and calls `UI_MainMenu` to push the initial screen. `UI_GetBotInfo` and arena loading in `ui_gameinfo.c` are invoked lazily on first access.

**Frame (`UI_REFRESH`):**
`vmMain` routes to `UI_Refresh` each client frame. It calls `UI_UpdateCvars` to sync cvar state, then draws the topmost menu on the stack by calling its registered `draw` callback (ultimately `Menu_Draw` → individual widget draw functions → `trap_R_*` calls). The connection screen (`UI_DrawConnectScreen`) is drawn directly when the client is in a connecting state, bypassing the menu stack.

**Input (`UI_KEY_EVENT`, `UI_MOUSE_EVENT`):**
Events are forwarded to the active menu's `key` callback via `Menu_DefaultKey`, which dispatches to focused widget handlers or fires `QM_ACTIVATED` events back to menu-specific `Event` functions. `K_CHAR_FLAG` multiplexes character events over the key-event path.

**Shutdown (`UI_SHUTDOWN`):**
`vmMain` routes to `UI_Shutdown`; the module releases no persistent allocations (all assets are managed by the engine's renderer/audio caches).

## Notable Implementation Details

- **Virtual coordinate space:** All drawing is performed in a fixed 640×480 logical space scaled to the actual screen resolution; coordinates are not inferable from the source to be hardware-dependent.
- **`UI_API_VERSION` override:** `ui_local.h` explicitly redefines `UI_API_VERSION` to `4`, overriding the value imported from `ui/ui_public.h`, to lock the q3_ui VM ABI version independently of the newer UI module.
- **`K_CHAR_FLAG` multiplexing:** Character input events are OR'd with `K_CHAR_FLAG` (1024) and sent through the same key-event path as scan-code events; menu handlers test for this bit to distinguish typed characters from navigation keys.
- **Menu stack isolation:** The UI maintains its own push-down menu stack (`UI_PushMenu`/`UI_PopMenu`) independent of the engine's console/chat input state; `UI_ForceMenuOff` clears the entire stack unconditionally.
- **Pool-allocated arena/bot metadata:** `ui_gameinfo.c` parses `.arena` and `.bot` script files into a flat pool allocator (`UI_Alloc`), with ordered index arrays built separately for SP, special, and FFA arena categories.
- **3D player previews gated on memory:** `ui_playermodel.c` guards live 3D player rendering behind a `LOW_MEMORY` (5 MB) threshold, falling back to static portrait bitmaps on low-memory systems.
- **GRank integration:** Several menus (`ui_login.c`, `ui_signup.c`, `ui_rankings.c`, `ui_rankstatus.c`, `ui_specifyleague.c`) integrate the optional GRank online ranking backend via non-standard `trap_CL_UI_Rank*` syscalls that are not declared in the bundled `ui_public.h` header, indicating they are GRank-specific extensions to the standard VM ABI.
- **Wizard-style server launch:** `ui_startserver.c` implements a three-stage wizard (map selection → server options → bot selection) as three independent `menuframework_s` structures navigated by shared event callbacks, with the final stage building and executing a `map` + `addbot` command sequence.
- **`punkbuster_items[]` cross-file coupling:** `ui_startserver.c` declares `punkbuster_items[]` as an `extern`, with the definition residing in `ui_servers2.c`, creating a direct link-time dependency between the two server-related menu files.
