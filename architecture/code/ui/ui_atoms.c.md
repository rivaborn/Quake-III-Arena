# code/ui/ui_atoms.c

## File Purpose
Provides foundational UI utility functions for the Quake III Arena menu system, including drawing primitives, coordinate scaling, console command dispatch, and post-game score tracking/persistence.

## Core Responsibilities
- Bridges `q_shared.c` error/print functions to UI trap calls (when not hard-linked)
- Scales 640×480 virtual coordinates to actual screen resolution
- Dispatches UI console commands (`postgame`, `ui_cache`, `remapShader`, etc.)
- Persists and loads per-map post-game best scores to/from `.game` files
- Provides primitive 2D drawing helpers (filled rects, outlines, named/handle pics)
- Manages the `m_entersound` flag for menu interaction audio

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `postGameInfo_t` | struct (typedef) | Holds end-of-match statistics: score, accuracy, bonuses, team scores, time |
| `uiStatic_t` | struct (typedef) | Old-UI static display context: resolution scale, shader handles, cursor, GL config |
| `uiInfo_t` | struct (typedef) | New-UI master state: map lists, server browser, player/team data, score info |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `m_entersound` | `qboolean` | global | Set after a frame to trigger menu-enter sound without disrupting caching |
| `newUI` | `qboolean` | global | Flag distinguishing old vs. new UI code path |

## Key Functions / Methods

### UI_AdjustFrom640
- **Signature:** `void UI_AdjustFrom640( float *x, float *y, float *w, float *h )`
- **Purpose:** Converts 640×480 virtual coordinates to actual screen pixels using stored scale factors.
- **Inputs:** Pointers to x, y, width, height in virtual space.
- **Outputs/Return:** Modifies all four values in-place.
- **Side effects:** None.
- **Calls:** None.
- **Notes:** Uses `uiInfo.uiDC.xscale` / `uiInfo.uiDC.yscale`; an older scale+bias path is commented out.

### UI_ConsoleCommand
- **Signature:** `qboolean UI_ConsoleCommand( int realTime )`
- **Purpose:** Engine-facing entry point for handling UI console commands each frame.
- **Inputs:** `realTime` — current engine time in ms.
- **Outputs/Return:** `qtrue` if command was handled, `qfalse` otherwise.
- **Side effects:** Updates `uiInfo.uiDC.frameTime` and `uiInfo.uiDC.realTime`; may trigger post-game flow, shader remapping, or cache operations.
- **Calls:** `UI_Argv`, `UI_ShowPostGame`, `UI_Report`, `UI_Load`, `trap_R_RemapShader`, `UI_CalcPostGameStats`, `UI_Cache_f`, `Q_stricmp`, `trap_Argc`.

### UI_CalcPostGameStats (static)
- **Signature:** `static void UI_CalcPostGameStats()`
- **Purpose:** Reads match result from command arguments, computes bonuses, compares against stored best, writes new high score to disk, and restores overridden cvars.
- **Inputs:** Command arguments via `UI_Argv` (indices 3–14).
- **Outputs/Return:** None.
- **Side effects:** File I/O to `games/<map>_<gametype>.game`; sets many `ui_score*` cvars; sets `uiInfo.newHighScoreTime`/`newBestTime`; calls `UI_SetBestScores`, `UI_ShowPostGame`.
- **Calls:** `trap_GetConfigString`, `trap_FS_FOpenFile`, `trap_FS_Read/Write/FCloseFile`, `trap_Cvar_VariableValue`, `trap_Cvar_Set`, `UI_Argv`, `UI_SetBestScores`, `UI_ShowPostGame`.

### UI_SetBestScores
- **Signature:** `void UI_SetBestScores(postGameInfo_t *newInfo, qboolean postGame)`
- **Purpose:** Pushes all fields of a `postGameInfo_t` into their corresponding `ui_score*` cvars; if `postGame` is true, also sets the `*2` variants for UI display.
- **Inputs:** `newInfo` — score data; `postGame` — whether to also populate secondary cvar set.
- **Outputs/Return:** None.
- **Side effects:** Writes ~15–30 cvars via `trap_Cvar_Set`.

### UI_LoadBestScores
- **Signature:** `void UI_LoadBestScores(const char *map, int game)`
- **Purpose:** Loads stored best-score data for a map/gametype from disk and checks demo availability.
- **Inputs:** `map` — map name string; `game` — game type integer.
- **Side effects:** File reads; sets `uiInfo.demoAvailable`; calls `UI_SetBestScores`.

### UI_DrawHandlePic
- **Signature:** `void UI_DrawHandlePic( float x, float y, float w, float h, qhandle_t hShader )`
- **Purpose:** Draws a shader handle as a stretched picture; supports negative w/h for horizontal/vertical flipping.
- **Side effects:** Calls `UI_AdjustFrom640`, `trap_R_DrawStretchPic`.

### UI_FillRect / UI_DrawRect
- **Purpose:** Draw filled or outlined rectangles in 640×480 virtual coordinates using the white shader.
- **Notes:** `UI_DrawRect` delegates to `UI_DrawTopBottom` + `UI_DrawSides`; both set/clear render color via `trap_R_SetColor`.

## Control Flow Notes
`UI_ConsoleCommand` is the primary engine-facing per-command entry point called by the client. Drawing helpers (`UI_FillRect`, `UI_DrawHandlePic`, etc.) are called during the UI refresh pass. Score persistence functions are triggered by the `postgame` server command. `UI_Shutdown` is a stub with no body.

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `keycodes.h`, `bg_public.h`, `ui_shared.h`
- `trap_*` functions — all defined in `ui_syscalls.c`; bridge to engine VM syscalls
- `uiInfo` (global `uiInfo_t`) — defined in `ui_main.c`
- `Display_CacheAll` — defined in `ui_shared.c`
- `UI_ShowPostGame`, `UI_Report`, `UI_Load` — defined in `ui_main.c`
- `Com_sprintf`, `Q_strncpyz`, `Q_stricmp`, `Info_ValueForKey` — defined in `q_shared.c`
