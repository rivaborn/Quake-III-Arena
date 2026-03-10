# code/q3_ui/ui_specifyleague.c

## File Purpose
Implements the "Specify League" UI menu for Quake III Arena's Global Rankings system, allowing players to enter a username, query available leagues for that player, and select one to set as the active `sv_leagueName` cvar.

## Core Responsibilities
- Initialize and lay out the Specify League menu screen with decorative bitmaps, a player name text field, a scrollable league list, and navigation buttons
- Query the Global Rankings backend for leagues associated with a given player name via `trap_CL_UI_RankGetLeauges`
- Populate a fixed-size list box with league names retrieved from numbered cvars (`leaguename1`, `leaguename2`, …)
- Re-query the league list when the player name field loses focus and the name has changed
- Write the selected league name to `sv_leagueName` cvar on back/confirm
- Pre-cache all required UI art shaders via `SpecifyLeague_Cache`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `specifyleague_t` | struct | Aggregates all menu widget instances (banner, frames, field, list, buttons) for the league selection screen |
| `table_t` | struct | Holds a display buffer (`buff`, 40 chars) and full league name (`leaguename`, 80 chars) for one list entry |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_specifyleague` | `specifyleague_t` | static (file) | Singleton menu state for this screen |
| `playername` | `char[80]` | static (file) | Cached player name used to detect when a re-query is needed |
| `league_table` | `table_t[128]` | file-static (non-`static`) | Storage for up to 128 league entries |
| `leaguename_items` | `char*[128]` | file-static (non-`static`) | Pointer array into `league_table[i].buff`; passed as `menulist_s.itemnames` |
| `specifyleague_artlist` | `static char*[]` | static (file) | NULL-terminated list of art paths to pre-cache |

## Key Functions / Methods

### SpecifyLeague_GetList
- **Signature:** `static void SpecifyLeague_GetList(void)`
- **Purpose:** Reads the current player name from the input field, calls the rankings trap to fetch leagues, then reads each result from numbered cvars into `league_table`.
- **Inputs:** Implicitly reads `s_specifyleague.rankname.field.buffer`
- **Outputs/Return:** `void`; populates `league_table[]` and sets `s_specifyleague.list.numitems`
- **Side effects:** Writes `playername`; reads cvars `leaguename1`…`leaguenameN` set by the trap call
- **Calls:** `Q_strncpyz`, `trap_CL_UI_RankGetLeauges`, `va`, `trap_Cvar_VariableStringBuffer`
- **Notes:** `trap_CL_UI_RankGetLeauges` is a Global Rankings-specific syscall not present in vanilla Q3; typo "Leauges" matches original source.

### SpecifyLeague_Event
- **Signature:** `static void SpecifyLeague_Event(void *ptr, int event)`
- **Purpose:** Central event dispatcher for all interactive widgets on the menu.
- **Inputs:** `ptr` — pointer to the triggering `menucommon_s`; `event` — `QM_ACTIVATED`, `QM_GOTFOCUS`, or `QM_LOSTFOCUS`
- **Outputs/Return:** `void`
- **Side effects:** May call `ScrollList_Key` (mutates list scroll state), `SpecifyLeague_GetList` (re-queries leagues), `trap_Cvar_Set("sv_leagueName", …)`, `UI_PopMenu`
- **Calls:** `ScrollList_Key`, `Q_strncmp`, `SpecifyLeague_GetList`, `trap_Cvar_Set`, `UI_PopMenu`
- **Notes:** Re-query triggers only on `QM_LOSTFOCUS` for the name field when content differs from cached `playername`. The `ID_SPECIFYLEAGUELIST / QM_GOTFOCUS` branch is a no-op stub (commented-out picture update).

### SpecifyLeague_MenuInit
- **Signature:** `void SpecifyLeague_MenuInit(void)`
- **Purpose:** Zeroes menu state, configures all widget positions/sizes/flags, links item pointers, pre-populates name field from the `name` cvar, and performs initial league query.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Writes all fields of `s_specifyleague`; initializes `league_table` and `leaguename_items`; calls `SpecifyLeague_GetList`
- **Calls:** `memset`, `SpecifyLeague_Cache`, `Menu_AddItem`, `Q_strncpyz`, `UI_Cvar_VariableString`, `SpecifyLeague_GetList`
- **Notes:** `grletters` bitmap is initialized in the struct declaration but never added via `Menu_AddItem`.

### SpecifyLeague_Cache
- **Signature:** `void SpecifyLeague_Cache(void)`
- **Purpose:** Pre-registers all art assets listed in `specifyleague_artlist` with the renderer.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for each path
- **Calls:** `trap_R_RegisterShaderNoMip`

### UI_SpecifyLeagueMenu
- **Signature:** `void UI_SpecifyLeagueMenu(void)`
- **Purpose:** Public entry point — initializes and pushes the menu onto the UI stack.
- **Calls:** `SpecifyLeague_MenuInit`, `UI_PushMenu`

## Control Flow Notes
`UI_SpecifyLeagueMenu` is called from elsewhere in the UI to open the screen. Menu interaction flows through `SpecifyLeague_Event` callbacks registered on each widget. On back/confirm (`ID_SPECIFYLEAGUEBACK`), the selected league from `league_table[list.curvalue]` is written to `sv_leagueName` and `UI_PopMenu` returns to the previous screen. No per-frame draw or update hook is registered; standard `Menu_Draw` handles rendering.

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `tr_types.h`, `ui_public.h`, `bg_public.h`, `keycodes.h`
- `trap_CL_UI_RankGetLeauges` — Global Rankings syscall, defined in `ui_syscalls.c` / engine; not declared in the bundled header (likely a GRank extension)
- `trap_Cvar_Set`, `trap_Cvar_VariableStringBuffer`, `trap_R_RegisterShaderNoMip` — engine syscalls declared in `ui_local.h`
- `Menu_AddItem`, `ScrollList_Key`, `UI_PushMenu`, `UI_PopMenu` — defined in `ui_qmenu.c` / `ui_atoms.c`
- `Q_strncpyz`, `Q_strncmp`, `va` — string utilities from `q_shared.c`
