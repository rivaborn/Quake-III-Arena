# code/q3_ui/ui_startserver.c

## File Purpose
Implements three interconnected UI menus for launching a multiplayer or single-player server: the **Start Server** map-selection menu, the **Server Options** configuration menu, and the **Bot Select** picker menu. Together they form a wizard-style flow: pick a map → configure options/bots → execute the server launch.

## Core Responsibilities
- Display a paginated grid of level-shot thumbnails filtered by game type for map selection
- Allow game type selection (FFA, Team DM, Tournament, CTF) and re-filter the map list accordingly
- Provide server configuration controls: frag/time/capture limits, friendly fire, pure server, dedicated mode, hostname, bot skill, PunkBuster
- Manage up to 12 player slots as Open/Bot/Closed with optional team assignment
- Display a paginated bot portrait grid for bot selection, sorted alphabetically
- Build and execute the `map` command along with `addbot` and `team` commands to start the server

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `startserver_t` | struct | State for the map-selection menu: paginated map list, gametype spinner, map thumbnail bitmaps, navigation |
| `serveroptions_t` | struct | State for the server options menu: limit fields, player slot arrays, bot names, dedicated/hostname/punkbuster controls |
| `botSelectInfo_t` | struct | State for the bot selection menu: sorted bot index array, paginated portrait grid, selected model tracking |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_startserver` | `startserver_t` | static (file) | Persistent state for the Start Server menu |
| `s_serveroptions` | `serveroptions_t` | static (file) | Persistent state for the Server Options menu |
| `botSelectInfo` | `botSelectInfo_t` | static (file) | Persistent state for the Bot Select menu |
| `gametype_items` | `const char *[]` | static (file) | Display strings for the game type spinner |
| `gametype_remap` | `int[]` | static (file) | Maps spinner index → `GT_*` enum value |
| `gametype_remap2` | `int[]` | static (file) | Maps `GT_*` value → `gametype_items` index (for display in options menu) |

## Key Functions / Methods

### GametypeBits
- **Signature:** `static int GametypeBits( char *string )`
- **Purpose:** Parses a space-separated type string (e.g. `"ffa team"`) and returns a bitmask of matching `GT_*` flags.
- **Inputs:** Arena info `"type"` value string.
- **Outputs/Return:** Integer bitmask (`1 << GT_FFA`, etc.).
- **Side effects:** None.
- **Calls:** `COM_ParseExt`, `Q_stricmp`
- **Notes:** Used both at cache time and at gametype-change time to populate the filtered map list.

### StartServer_GametypeEvent
- **Signature:** `static void StartServer_GametypeEvent( void* ptr, int event )`
- **Purpose:** Rebuilds the filtered map list whenever the game type spinner changes; resets page and selection to 0.
- **Inputs:** Menu callback pointer; `QM_ACTIVATED` event required to act.
- **Outputs/Return:** void; mutates `s_startserver.maplist`, `nummaps`, `maxpages`, `page`, `currentmap`.
- **Side effects:** Calls `StartServer_Update()` to refresh display.
- **Calls:** `UI_GetNumArenas`, `UI_GetArenaInfoByNumber`, `GametypeBits`, `Info_ValueForKey`, `Q_strncpyz`, `Q_strupr`, `StartServer_Update`

### StartServer_Update
- **Signature:** `static void StartServer_Update( void )`
- **Purpose:** Syncs the visible map thumbnail widgets and map-name label to the current page and selected map.
- **Inputs:** None (reads `s_startserver` globals).
- **Outputs/Return:** void.
- **Side effects:** Modifies `QMF_HIGHLIGHT`, `QMF_INACTIVE`, `QMF_PULSEIFFOCUS` flags on `mappics[]` and `mapbuttons[]`; updates `mapname.string`.
- **Calls:** `Com_sprintf`, `strcpy`, `Q_strupr`

### ServerOptions_Start
- **Signature:** `static void ServerOptions_Start( void )`
- **Purpose:** Reads all configured options, sets cvars, then issues `map`, `addbot`, and `team` console commands to launch the server.
- **Inputs:** None (reads `s_serveroptions` and `s_startserver`).
- **Outputs/Return:** void.
- **Side effects:** Sets many cvars (`sv_maxclients`, `dedicated`, `timelimit`, `fraglimit`, `capturelimit`, `g_friendlyfire`, `sv_pure`, `sv_hostname`, `sv_punkbuster`, per-gametype `ui_*` cvars); appends commands to the command buffer.
- **Calls:** `trap_Cvar_SetValue`, `trap_Cvar_Set`, `trap_Cmd_ExecuteText`, `Com_Clamp`, `Com_sprintf`, `atoi`
- **Notes:** Uses `EXEC_APPEND` + `wait` to ensure `dedicated` takes effect before `map` runs.

### StartServer_MenuInit / UI_StartServerMenu
- **Signature:** `static void StartServer_MenuInit(void)` / `void UI_StartServerMenu(qboolean multiplayer)`
- **Purpose:** Initializes and pushes the map-selection menu.
- **Side effects:** Calls `StartServer_Cache`, zeroes `s_startserver`, adds all menu items, fires the initial gametype event to populate the map list.

### ServerOptions_MenuInit / UI_ServerOptionsMenu
- **Signature:** `static void ServerOptions_MenuInit(qboolean multiplayer)` / `static void UI_ServerOptionsMenu(qboolean multiplayer)`
- **Purpose:** Initializes and pushes the server options menu; conditionally adds/omits widgets based on gametype and multiplayer flag.
- **Side effects:** Calls `ServerOptions_Cache`, `ServerOptions_SetMenuItems` (which seeds bot names and player slot types).

### UI_BotSelectMenu_Init / UI_BotSelectMenu
- **Signature:** `static void UI_BotSelectMenu_Init(char *bot)` / `void UI_BotSelectMenu(char *bot)`
- **Purpose:** Initializes and pushes the bot selection menu, pre-selecting the named bot if found.
- **Side effects:** Calls `UI_BotSelectMenu_BuildList` (sorts bots), `UI_BotSelectMenu_Default`, `UI_BotSelectMenu_UpdateGrid`.

### UI_BotSelectMenu_SelectEvent
- **Signature:** `static void UI_BotSelectMenu_SelectEvent(void* ptr, int event)`
- **Purpose:** On Accept, pops the bot menu and writes the chosen bot name back into `s_serveroptions.newBotName`; sets `newBot = qtrue` as a deferred-update flag.
- **Side effects:** Mutates `s_serveroptions.newBot`, `s_serveroptions.newBotName`; calls `UI_PopMenu`.
- **Notes:** The actual name copy into `playerNameBuffers` is deferred to the next `ServerOptions_LevelshotDraw` frame to avoid race issues with the menu stack.

### StartServer_Cache / ServerOptions_Cache / UI_BotSelectMenu_Cache
- **Purpose:** Pre-register all shader assets for their respective menus.
- **Notes:** `StartServer_Cache` also pre-populates the full (unfiltered) map list and optionally precaches all levelshot shaders if `com_buildscript` is set.

## Control Flow Notes
All three menus are modal overlays managed by `UI_PushMenu`/`UI_PopMenu`. The flow is linear: `UI_StartServerMenu` → `UI_ServerOptionsMenu` (triggered by `ID_STARTSERVERNEXT`) → `UI_BotSelectMenu` (triggered per-slot). Server launch fires from `ServerOptions_Start` via `ID_GO`. These menus have no per-frame update hook; display updates are event-driven or occur inside `ownerdraw` callbacks called during `Menu_Draw`.

## External Dependencies
- **Includes:** `ui_local.h` (menu framework, trap syscalls, shared types)
- **Defined elsewhere:** `punkbuster_items[]` (extern from `ui_servers2.c`); `UI_ServerOptionsMenu` forward-declared static but called from `StartServer_MenuEvent`
- **Trap syscalls used:** `trap_R_RegisterShaderNoMip`, `trap_Cvar_SetValue`, `trap_Cvar_Set`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cmd_ExecuteText`
- **UI info queries:** `UI_GetNumArenas`, `UI_GetArenaInfoByNumber`, `UI_GetArenaInfoByMap`, `UI_GetNumBots`, `UI_GetBotInfoByNumber`, `UI_GetBotInfoByName`, `Info_ValueForKey`
