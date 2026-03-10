# code/ui/ui_gameinfo.c

## File Purpose
Loads and parses arena map and bot definition files (`.arena`, `.bot`, `arenas.txt`, `bots.txt`) into global UI-accessible arrays. Provides lookup functions for bot info strings and populates `uiInfo.mapList` with parsed arena metadata used by the UI menu system.

## Core Responsibilities
- Parse key-value info blocks from arena/bot text files via `UI_ParseInfos`
- Load all arena definitions from `scripts/arenas.txt` and `*.arena` files into `ui_arenaInfos[]`
- Load all bot definitions from `scripts/bots.txt` and `*.bot` files into `ui_botInfos[]`
- Populate `uiInfo.mapList[]` with map name, load name, image path, and game-type bitfields
- Provide bot lookup by index or name
- Respect `g_arenasFile` / `g_botsFile` cvars to override default paths

## Key Types / Data Structures

None defined in this file. Relies on `mapInfo` (from `ui_local.h`) and raw `char*` info strings.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `ui_numBots` | `int` | global | Count of loaded bot entries |
| `ui_botInfos` | `static char*[MAX_BOTS]` | static | Array of raw key-value bot info strings |
| `ui_numArenas` | `static int` | static | Count of loaded arena entries |
| `ui_arenaInfos` | `static char*[MAX_ARENAS]` | static | Array of raw key-value arena info strings |
| `ui_numSinglePlayerArenas` | `static int` | static (non-MISSIONPACK) | SP arena count |
| `ui_numSpecialSinglePlayerArenas` | `static int` | static (non-MISSIONPACK) | Special SP arena count |

## Key Functions / Methods

### UI_ParseInfos
- **Signature:** `int UI_ParseInfos( char *buf, int max, char *infos[] )`
- **Purpose:** Tokenizes a text buffer of `{ key value ... }` blocks into info strings stored in the provided array.
- **Inputs:** `buf` — raw file text; `max` — max entries to parse; `infos[]` — output pointer array
- **Outputs/Return:** Count of successfully parsed info blocks
- **Side effects:** Allocates heap memory via `UI_Alloc` for each info string
- **Calls:** `COM_Parse`, `COM_ParseExt`, `Q_strncpyz`, `Info_SetValueForKey`, `UI_Alloc`, `Com_Printf`
- **Notes:** Allocates extra space for a `\num\N` suffix to be added later; returns early on `{` mismatch or capacity exceeded

### UI_LoadArenasFromFile
- **Signature:** `static void UI_LoadArenasFromFile( char *filename )`
- **Purpose:** Opens a single arena file, reads it into a buffer, and appends parsed entries into `ui_arenaInfos`.
- **Inputs:** `filename` — VFS-relative path
- **Outputs/Return:** void; increments `ui_numArenas`
- **Side effects:** File I/O via trap syscalls; calls `UI_ParseInfos`
- **Calls:** `trap_FS_FOpenFile`, `trap_FS_Read`, `trap_FS_FCloseFile`, `trap_Print`, `UI_ParseInfos`
- **Notes:** Buffer capped at `MAX_ARENAS_TEXT`; prints red error on missing or oversized file

### UI_LoadArenas
- **Signature:** `void UI_LoadArenas( void )`
- **Purpose:** Orchestrates full arena loading from the primary file and all `.arena` files, then populates `uiInfo.mapList[]` with typed map metadata.
- **Inputs:** None
- **Outputs/Return:** void; fills `uiInfo.mapList` and `uiInfo.mapCount`
- **Side effects:** Registers `g_arenasFile` cvar; allocates strings via `String_Alloc`; modifies global `uiInfo`
- **Calls:** `trap_Cvar_Register`, `UI_LoadArenasFromFile`, `trap_FS_GetFileList`, `trap_Print`, `UI_OutOfMemory`, `Info_ValueForKey`, `String_Alloc`, `strstr`
- **Notes:** Game-type bits set by substring matching `type` field against `"ffa"`, `"tourney"`, `"ctf"`, `"oneflag"`, `"overload"`, `"harvester"`; unmapped type defaults to FFA; stops at `MAX_MAPS`

### UI_LoadBotsFromFile
- **Signature:** `static void UI_LoadBotsFromFile( char *filename )`
- **Purpose:** Opens a single bot file, compresses whitespace, and appends parsed entries into `ui_botInfos`.
- **Inputs:** `filename` — VFS-relative path
- **Outputs/Return:** void; increments `ui_numBots`
- **Side effects:** File I/O; calls `COM_Compress` on buffer before parsing
- **Calls:** `trap_FS_FOpenFile`, `trap_FS_Read`, `trap_FS_FCloseFile`, `trap_Print`, `COM_Compress`, `UI_ParseInfos`

### UI_LoadBots
- **Signature:** `void UI_LoadBots( void )`
- **Purpose:** Orchestrates full bot loading from the primary file and all `.bot` files.
- **Inputs/Outputs:** void / void; populates `ui_botInfos` and `ui_numBots`
- **Side effects:** Registers `g_botsFile` cvar
- **Calls:** `trap_Cvar_Register`, `UI_LoadBotsFromFile`, `trap_FS_GetFileList`, `trap_Print`

### UI_GetBotInfoByNumber / UI_GetBotInfoByName
- **Notes:** Simple index/linear-search accessors over `ui_botInfos[]`; `ByName` uses case-insensitive `Q_stricmp` on the `name` key; `ByNumber` returns `NULL` on out-of-range

### UI_GetBotNameByNumber
- **Signature:** `char *UI_GetBotNameByNumber( int num )`
- **Purpose:** Returns the `name` key string for a bot by index, defaulting to `"Sarge"` on failure.

## Control Flow Notes
`UI_LoadArenas` and `UI_LoadBots` are called during UI initialization (from `UI_Load` / `UI_InitGameinfo`). They run once at startup and populate the static arrays and `uiInfo` structures used throughout the menu lifetime. There is no per-frame involvement.

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_shared.h`
- **Defined elsewhere:** `COM_Parse`, `COM_ParseExt`, `COM_Compress`, `Info_SetValueForKey`, `Info_ValueForKey`, `Q_strncpyz`, `Q_stricmp`, `UI_Alloc`, `UI_OutOfMemory`, `String_Alloc`, all `trap_*` syscall wrappers, `uiInfo` global, `MAX_BOTS`, `MAX_ARENAS`, `MAX_ARENAS_TEXT`, `MAX_BOTS_TEXT`, `MAX_MAPS`, game-type enum constants (`GT_FFA`, etc.)
