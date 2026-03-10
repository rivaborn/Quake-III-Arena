# code/q3_ui/ui_gameinfo.c

## File Purpose
Manages loading, parsing, and querying arena and bot metadata for the Quake III Arena UI module. Also tracks and persists single-player game progression, award data, and tier video unlock state via cvars.

## Core Responsibilities
- Load and parse arena info from `.arena` files and `scripts/arenas.txt` into a pool allocator
- Load and parse bot info from `.bot` files and `scripts/bots.txt`
- Assign ordered indices to arenas, separating single-player, special, and FFA arenas
- Query arena/bot records by number, map name, or special tag
- Read and write single-player scores per skill level via `g_spScores1–5` cvars
- Track award totals and tier cinematic unlock state via cvars
- Provide cheat/debug commands to unlock all levels and medals

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `awardType_t` | enum (defined in `ui_local.h`) | Enumerates the six award categories used to key award cvar data |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `ui_numBots` | `int` | global | Count of loaded bot entries |
| `ui_botInfos` | `char*[MAX_BOTS]` | static | Pointers into pool for each bot info string |
| `ui_numArenas` | `int` | static | Count of loaded arena entries |
| `ui_arenaInfos` | `char*[MAX_ARENAS]` | static | Pointers into pool for each arena info string |
| `ui_numSinglePlayerArenas` | `int` | static | Count of non-special SP arenas (tier-aligned) |
| `ui_numSpecialSinglePlayerArenas` | `int` | static | Count of special SP arenas (training, final) |
| `memoryPool` | `char[128*1024]` | static | Fixed pool backing arena/bot string storage |
| `allocPoint` | `int` | static | Current allocation offset into `memoryPool` |
| `outOfMemory` | `int` | static | Flag set when pool is exhausted |

## Key Functions / Methods

### UI_Alloc
- **Signature:** `void *UI_Alloc( int size )`
- **Purpose:** Bump-pointer allocator drawing from `memoryPool`
- **Inputs:** `size` — bytes requested
- **Outputs/Return:** Pointer to allocated block, or `NULL` on overflow
- **Side effects:** Advances `allocPoint` (aligned to 32 bytes); sets `outOfMemory` on overflow
- **Calls:** None
- **Notes:** No free; allocation is reset only by `UI_InitMemory`. Alignment is `(size+31)&~31`.

### UI_ParseInfos
- **Signature:** `int UI_ParseInfos( char *buf, int max, char *infos[] )`
- **Purpose:** Parses a series of `{ key value ... }` blocks from a text buffer into info strings stored in the pool
- **Inputs:** `buf` — null-terminated text; `max` — max entries to add; `infos[]` — output pointer array
- **Outputs/Return:** Number of entries parsed
- **Side effects:** Calls `UI_Alloc` per entry; modifies `buf` (COM_Parse advances the pointer)
- **Calls:** `COM_Parse`, `COM_ParseExt`, `Q_strncpyz`, `Info_SetValueForKey`, `UI_Alloc`, `Com_Printf`
- **Notes:** Allocates extra space for a `\num\NNN` suffix appended later by callers.

### UI_LoadArenas
- **Signature:** `static void UI_LoadArenas( void )`
- **Purpose:** Orchestrates loading of all arena definitions, assigns sequential `num` keys, counts SP vs. special arenas, and realigns the SP count to a `ARENAS_PER_TIER` boundary
- **Inputs:** None (reads `g_arenasFile` cvar and filesystem)
- **Outputs/Return:** void; populates `ui_arenaInfos`, `ui_numArenas`, `ui_numSinglePlayerArenas`, `ui_numSpecialSinglePlayerArenas`
- **Side effects:** Filesystem reads; cvar registration; calls `trap_Print`
- **Calls:** `trap_Cvar_Register`, `UI_LoadArenasFromFile`, `trap_FS_GetFileList`, `Info_SetValueForKey`, `Info_ValueForKey`, `trap_Print`

### UI_GetArenaInfoByNumber
- **Signature:** `const char *UI_GetArenaInfoByNumber( int num )`
- **Purpose:** Linear scan returning the arena info string whose `num` key equals the requested index
- **Inputs:** `num` — logical arena number
- **Outputs/Return:** Pointer to info string, or `NULL`
- **Side effects:** `trap_Print` on invalid range
- **Notes:** O(n) scan; arena `num` keys are not stored in sorted array order.

### UI_GetBestScore / UI_SetBestScore
- **Signature:** `void UI_GetBestScore( int level, int *score, int *skill )` / `void UI_SetBestScore( int level, int score )`
- **Purpose:** Read/write best finish placement (1–8) for a level across all five skill bands stored as info strings in `g_spScores1–5` cvars
- **Inputs:** `level` index; `score` placement; `skill` band (1–5)
- **Side effects:** Reads/writes cvars via `trap_Cvar_VariableStringBuffer` / `trap_Cvar_Set`

### UI_TierCompleted
- **Signature:** `int UI_TierCompleted( int levelWon )`
- **Purpose:** After winning a level, checks whether its tier (or the final/training special arenas) is fully completed at rank 1
- **Outputs/Return:** Tier number + 1 if completed, 0 for training, -1 if not yet done
- **Calls:** `UI_GetNumSPTiers`, `UI_GetSpecialArenaInfo`, `UI_GetBestScore`, `Info_ValueForKey`

### UI_InitGameinfo
- **Signature:** `void UI_InitGameinfo( void )`
- **Purpose:** Entry point called at UI startup; resets pool, loads arenas and bots, and sets `uis.demoversion`
- **Calls:** `UI_InitMemory`, `UI_LoadArenas`, `UI_LoadBots`, `trap_Cvar_VariableValue`

## Control Flow Notes
`UI_InitGameinfo` is called once during UI initialization (from `ui_main.c`). After that, all functions are query-only or cvar read/write, invoked on demand by menu screens (SP level selection, postgame, award screens). `UI_SPUnlock_f` and `UI_SPUnlockMedals_f` are registered as console commands.

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `ui_public.h` — shared types, `vmCvar_t`, `qboolean`, info-string API
- `COM_Parse`, `COM_ParseExt` — defined in `qcommon`
- `Info_SetValueForKey`, `Info_ValueForKey` — defined in `q_shared.c`
- `trap_*` syscalls — defined in `ui_syscalls.c`, dispatched into the engine VM interface
- `UI_SPLevelMenu_ReInit` — defined in `ui_spLevel.c`
- `uis` (`uiStatic_t`) — global UI state defined in `ui_atoms.c`
