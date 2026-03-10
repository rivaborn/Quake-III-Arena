# code/botlib/be_interface.c

## File Purpose
This is the primary entry point and export layer for Quake III Arena's bot library (botlib). It initializes, shuts down, and exposes all bot subsystem APIs to the engine via a versioned function-pointer table (`botlib_export_t`), and bridges the engine's import callbacks into the library's global `botimport`.

## Core Responsibilities
- Implement `GetBotLibAPI`, the single DLL/module entry point that validates API version and returns `botlib_export_t`
- Initialize all bot subsystems in order (AAS, EA, WeaponAI, GoalAI, ChatAI, MoveAI) via `Export_BotLibSetup`
- Shut down all subsystems and free all resources via `Export_BotLibShutdown`
- Validate client/entity numbers before forwarding calls to internal subsystems
- Populate the three nested export structs (AAS, EA, AI) with function pointers to internal implementations
- Expose libvar get/set, precompiler handle functions, frame ticking, and map loading
- Provide a debug-only `BotExportTest` hook for in-engine AAS visualization

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `botlib_globals_t` | struct (defined in `be_interface.h`) | Holds runtime globals: `maxclients`, `maxentities`, `botlibsetup`, `goalareanum`, `goalorigin`, etc. |
| `botlib_export_t` | struct (defined in `botlib.h`) | Versioned function-pointer table returned to the engine; contains nested `aas`, `ea`, `ai` sub-tables |
| `botlib_import_t` | struct (defined in `botlib.h`) | Engine callbacks imported at startup (Print, DebugLineCreate, etc.) |
| `aas_export_t` | struct | Sub-table of AAS spatial query functions |
| `ea_export_t` | struct | Sub-table of elementary bot action functions |
| `ai_export_t` | struct | Sub-table of higher-level AI functions (char, chat, goal, move, weapon, gen) |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `botlibglobals` | `botlib_globals_t` | global | Central runtime state for the library (client/entity limits, setup flag, debug goal) |
| `be_botlib_export` | `botlib_export_t` | global | The export table returned by `GetBotLibAPI`; pointer to it is the library's public API handle |
| `botimport` | `botlib_import_t` | global | Engine-provided callbacks; used throughout botlib for printing, debug rendering, etc. |
| `bot_developer` | `int` | global | Non-zero when `bot_developer` libvar is set; gates verbose output |
| `botlibsetup` | `int` | global | Redundant setup flag (mirrors `botlibglobals.botlibsetup`); guards against calls before init |

## Key Functions / Methods

### GetBotLibAPI
- **Signature:** `botlib_export_t *GetBotLibAPI(int apiVersion, botlib_import_t *import)`
- **Purpose:** Module entry point. Validates API version, stores the engine import table, then populates and returns `be_botlib_export`.
- **Inputs:** `apiVersion` — must equal `BOTLIB_API_VERSION`; `import` — engine callbacks
- **Outputs/Return:** Pointer to `be_botlib_export`, or `NULL` on version mismatch
- **Side effects:** Writes `botimport`; zeroes `be_botlib_export`; calls `Init_AAS_Export`, `Init_EA_Export`, `Init_AI_Export`
- **Calls:** `Init_AAS_Export`, `Init_EA_Export`, `Init_AI_Export`, `botimport.Print`
- **Notes:** Asserts `import` and `import->Print` are non-null.

### Export_BotLibSetup
- **Signature:** `int Export_BotLibSetup(void)`
- **Purpose:** Initializes all bot subsystems sequentially; aborts on first error.
- **Inputs:** None (reads libvars `bot_developer`, `maxclients`, `maxentities`)
- **Outputs/Return:** `BLERR_NOERROR` or a subsystem error code
- **Side effects:** Zeroes `botlibglobals`; opens `botlib.log`; initializes AAS, EA, WeaponAI, GoalAI, ChatAI, MoveAI; sets `botlibsetup = qtrue`
- **Calls:** `LibVarGetValue`, `LibVarValue`, `Log_Open`, `AAS_Setup`, `EA_Setup`, `BotSetupWeaponAI`, `BotSetupGoalAI`, `BotSetupChatAI`, `BotSetupMoveAI`

### Export_BotLibShutdown
- **Signature:** `int Export_BotLibShutdown(void)`
- **Purpose:** Tears down all subsystems in reverse order; frees libvars, precompiler defines, and closes log.
- **Inputs:** None
- **Outputs/Return:** `BLERR_NOERROR` or `BLERR_LIBRARYNOTSETUP`
- **Side effects:** Calls shutdown on all AI subsystems, AAS, EA; frees all libvars and global precompiler defines; closes log file; sets `botlibsetup = qfalse`
- **Calls:** `BotShutdownChatAI`, `BotShutdownMoveAI`, `BotShutdownGoalAI`, `BotShutdownWeaponAI`, `BotShutdownWeights`, `BotShutdownCharacters`, `AAS_Shutdown`, `EA_Shutdown`, `LibVarDeAllocAll`, `PC_RemoveAllGlobalDefines`, `Log_Shutdown`, `PC_CheckOpenSourceHandles`

### Export_BotLibLoadMap
- **Signature:** `int Export_BotLibLoadMap(const char *mapname)`
- **Purpose:** Loads AAS data for a new map and initializes goal and movement subsystems for that level.
- **Inputs:** `mapname` — map filename
- **Outputs/Return:** `BLERR_NOERROR` or AAS load error code
- **Side effects:** Calls `AAS_LoadMap`, `BotInitLevelItems`, `BotSetBrushModelTypes`; prints timing in DEBUG builds
- **Calls:** `BotLibSetup`, `AAS_LoadMap`, `BotInitLevelItems`, `BotSetBrushModelTypes`, `botimport.Print`

### Init_AAS_Export / Init_EA_Export / Init_AI_Export
- **Signature:** `static void Init_*_Export(<table_type> *)`
- **Purpose:** Populate the respective sub-tables with function pointers to internal implementations.
- **Notes:** Pure assignment; no logic. Cover all public AAS spatial queries, all EA bot-action primitives, and all high-level AI (char, chat, goal, move, weapon, gen) functions.

### BotExportTest
- **Signature:** `int BotExportTest(int parm0, char *parm1, vec3_t parm2, vec3_t parm3)`
- **Purpose:** Debug-only testing hook exposed as `be_botlib_export.Test`; drives AAS visualization from a player position.
- **Side effects:** Renders AAS area polygons, reachability paths, and debug lines via `botimport`; modifies static local state (`area`, `line[]`).
- **Notes:** Entire body is `#ifdef DEBUG`-guarded; always returns 0 in release builds.

## Control Flow Notes
- `GetBotLibAPI` is called once at botlib load time (called from `sv_bot.c` or equivalent).
- `Export_BotLibSetup` / `Export_BotLibShutdown` map to server-side `BotLibSetup`/`BotLibShutdown` calls.
- `Export_BotLibStartFrame` is called every server frame, forwarding to `AAS_StartFrame`.
- `Export_BotLibUpdateEntity` is called per-entity per-frame to sync world state into AAS.
- Map transitions trigger `Export_BotLibLoadMap`.

## External Dependencies
- `../game/botlib.h` — `botlib_export_t`, `botlib_import_t`, `BOTLIB_API_VERSION`, error codes
- `../game/be_aas.h`, `be_aas_funcs.h`, `be_aas_def.h` — AAS query functions and `aasworld` global
- `../game/be_ea.h` — Elementary action functions
- `../game/be_ai_*.h` — Higher-level AI subsystem APIs
- `be_ai_weight.h` — `BotShutdownWeights`
- `l_libvar.h` — Library variable system
- `l_precomp.h` — Precompiler (PC_*) functions; defined in `l_precomp.c`
- `l_log.h` — Log file; defined in `l_log.c`
- `aasworld` (global `aas_t`) — Defined in `be_aas_main.c`/`be_aas_def.h`; accessed directly in `BotExportTest`
