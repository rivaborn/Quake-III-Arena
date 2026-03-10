# code/ui/ui_main.c

## File Purpose
The primary entry point and master controller for Quake III Arena's Team Arena UI module. It implements the `vmMain` dispatch function (the QVM entry point), manages all menu data, handles owner-draw rendering, input routing, server browser logic, and asset lifecycle for the entire UI system.

## Core Responsibilities
- Dispatch all UI VM commands via `vmMain` (init, shutdown, key/mouse events, refresh, active menu)
- Initialize and wire the `displayContextDef_t` function table with UI callbacks during `_UI_Init`
- Render per-frame UI: paint menus, draw cursor, update server/player lists via `_UI_Refresh`
- Implement all owner-draw items (handicap, player model, clan logo, map preview, team slots, etc.)
- Manage server browser: refresh, display list construction, binary insertion sorting, find-player searches
- Parse game data files: `gameinfo.txt`, `teaminfo.txt`, map lists, game types, character/alias tables
- Register and update all UI cvars through a static `cvarTable[]` descriptor array
- Execute menu scripts (`UI_RunMenuScript`) for game start, server join, bot add, settings changes

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `uiInfo_t` | struct (typedef) | Master UI state: all lists (maps, teams, characters, servers, mods, demos), server status, cinematic handles, player info |
| `cvarTable_t` | struct (typedef) | Descriptor binding a `vmCvar_t*` to its cvar name, default value, and flags |
| `serverFilter_t` | struct (typedef) | Pairs a display description with a game basedir for server filtering |
| `serverStatusCvar_t` | struct (typedef) | Maps raw cvar key names to human-readable alt names for server status display |
| `playerInfo_t` | struct (typedef, in header) | Full skeletal animation state for rendering a player model in the UI |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `uiInfo` | `uiInfo_t` | global | Singleton holding all UI runtime state |
| `ui_new`, `ui_debug`, `ui_initialized`, `ui_teamArenaFirstRun` | `vmCvar_t` | global | Core UI control cvars |
| `frameCount`, `startTime` | `int` | global | Frame counting (partially unused) |
| `updateModel` | `static qboolean` | file-static | Dirty flag triggering player model rebuild |
| `q3Model` | `static qboolean` | file-static | Tracks whether Q3 model mode is active vs. team model |
| `updateOpponentModel` | `static qboolean` | file-static | Dirty flag for opponent model rebuild |
| `lastConnState`, `lastLoadingText` | static | file-static | Track connection state changes in connect screen |
| `cvarTable[]` | `static cvarTable_t[]` | file-static | Full descriptor table for all UI cvars |
| `defaultMenu` | `char*` | global | Fallback menu buffer pointer |

## Key Functions / Methods

### vmMain
- **Signature:** `int vmMain(int command, int arg0..arg11)`
- **Purpose:** QVM entry point; the only way the engine calls into the UI module
- **Inputs:** `command` enum (UI_INIT, UI_SHUTDOWN, UI_REFRESH, UI_KEY_EVENT, etc.), up to 12 integer args
- **Outputs/Return:** Return value depends on command; -1 for unknown
- **Side effects:** Dispatches to all other subsystems
- **Calls:** `_UI_Init`, `_UI_Shutdown`, `_UI_KeyEvent`, `_UI_MouseEvent`, `_UI_Refresh`, `_UI_IsFullscreen`, `_UI_SetActiveMenu`, `UI_ConsoleCommand`, `UI_DrawConnectScreen`

---

### _UI_Init
- **Signature:** `void _UI_Init(qboolean inGameLoad)`
- **Purpose:** Full UI subsystem initialization; wires the display context function table, registers cvars, parses game data, loads menus and assets
- **Inputs:** `inGameLoad` — whether loading from in-game (affects menu set selection)
- **Outputs/Return:** void
- **Side effects:** Populates `uiInfo.uiDC` function pointers, calls `AssetCache`, `UI_ParseTeamInfo`, `UI_ParseGameInfo`, `UI_LoadMenus`, `UI_BuildQ3Model_List`, `UI_LoadBots`, initializes cinematic handles to -1
- **Calls:** `UI_RegisterCvars`, `UI_InitMemory`, `trap_GetGlconfig`, `AssetCache`, `Init_Display`, `String_Init`, `UI_ParseTeamInfo`, `UI_LoadTeams`, `UI_ParseGameInfo`, `UI_LoadMenus`, `trap_LAN_LoadCachedServers`, `UI_LoadBestScores`, `UI_BuildQ3Model_List`, `UI_LoadBots`

---

### _UI_Refresh
- **Signature:** `void _UI_Refresh(int realtime)`
- **Purpose:** Per-frame UI update: advances time, computes FPS, repaints all menus, refreshes server data, draws cursor
- **Inputs:** `realtime` — current engine time in milliseconds
- **Side effects:** Writes `uiInfo.uiDC.frameTime`, `uiInfo.uiDC.realTime`, `uiInfo.uiDC.FPS`; calls rendering and server refresh functions
- **Calls:** `UI_UpdateCvars`, `Menu_PaintAll`, `UI_DoServerRefresh`, `UI_BuildServerStatus`, `UI_BuildFindPlayerList`, `UI_DrawHandlePic`

---

### UI_OwnerDraw
- **Signature:** `static void UI_OwnerDraw(float x, float y, float w, float h, float text_x, float text_y, int ownerDraw, int ownerDrawFlags, int align, float special, float scale, vec4_t color, qhandle_t shader, int textStyle)`
- **Purpose:** Central dispatch for all owner-draw element rendering; maps `ownerDraw` enum to specific draw functions
- **Side effects:** Calls all UI_Draw* subfunctions which may register shaders, run cinematics, set render colors

---

### UI_RunMenuScript
- **Signature:** `static void UI_RunMenuScript(char **args)`
- **Purpose:** Executes named menu script commands (StartServer, JoinServer, loadArenas, SkirmishStart, addBot, orders, etc.)
- **Inputs:** `args` — tokenized script argument stream
- **Side effects:** Calls `trap_Cmd_ExecuteText`, `trap_Cvar_Set`, starts/stops server refresh, modifies game state cvars, manipulates cinematic handles

---

### UI_BuildServerDisplayList
- **Signature:** `static void UI_BuildServerDisplayList(qboolean force)`
- **Purpose:** Filters and sorts the LAN server list into `uiInfo.serverStatus.displayServers` for the browser UI
- **Side effects:** Modifies `uiInfo.serverStatus.numDisplayServers`, calls `UI_BinaryServerInsertion`, `trap_LAN_MarkServerVisible`
- **Notes:** Uses binary insertion sort; `force==2` is a special "final pass" sentinel

---

### _UI_Init / Text_Paint / Text_Width / Text_Height / Text_PaintWithCursor
- **Notes:** Bitmap font rendering functions operating on `fontInfo_t` glyph tables; select small/normal/big font by scale threshold against `ui_smallFont` and `ui_bigFont` cvars; handle Q3 color escape codes inline

---

### UI_DrawConnectScreen
- **Signature:** `void UI_DrawConnectScreen(qboolean overlay)`
- **Purpose:** Renders the connection/loading screen; shows map name, server address, download progress, connection state text
- **Side effects:** Calls `Text_PaintCenter`, `UI_DisplayDownloadInfo`; reads cvars `cl_downloadName`, `cl_downloadSize`, etc.

---

### UI_ParseGameInfo / UI_ParseTeamInfo
- **Purpose:** Parse `gameinfo.txt` and `teaminfo.txt` data files into `uiInfo.gameTypes`, `uiInfo.mapList`, `uiInfo.teamList`, `uiInfo.characterList`, `uiInfo.aliasList` using the PC (script compiler) token stream
- **Side effects:** Populates global `uiInfo` lists; registers shaders for team icons

## Control Flow Notes
- **Init:** `vmMain(UI_INIT)` → `_UI_Init` — runs once at module load; sets up everything
- **Per-frame:** `vmMain(UI_REFRESH)` → `_UI_Refresh` — called every frame; drives menu painting and server polling
- **Input:** `vmMain(UI_KEY_EVENT/UI_MOUSE_EVENT)` → `_UI_KeyEvent/_UI_MouseEvent` — routes input to focused menu
- **Shutdown:** `vmMain(UI_SHUTDOWN)` → `_UI_Shutdown` → `trap_LAN_SaveCachedServers`
- **Cvar sync:** `UI_UpdateCvars` is called each `_UI_Refresh` tick to pull engine cvar values into `vmCvar_t` mirrors

## External Dependencies
- `ui_local.h` → `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `ui_shared.h`, `keycodes.h`
- **Defined elsewhere:** `Menu_Count`, `Menu_PaintAll`, `Menu_GetFocused`, `Menu_HandleKey`, `Menu_New`, `Menu_Reset`, `Menu_SetFeederSelection`, `Menus_*`, `Display_*`, `Init_Display`, `String_*`, `Controls_*`, `UI_DrawPlayer`, `UI_PlayerInfo_*`, `UI_RegisterClientModelname`, `UI_LoadArenas`, `UI_LoadBestScores`, `UI_ClearScores`, `UI_LoadBots`, `UI_GetBotNameByNumber`, `UI_GetNumBots`, `trap_*` syscalls (all defined in `ui_syscalls.c`)
