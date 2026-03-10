# code/cgame/cg_consolecmds.c

## File Purpose
Registers and dispatches client-side console commands typed at the local console or bound to keys. It bridges player input (keyboard bindings, console text) to cgame actions such as score display, weapon cycling, team orders, and voice chat.

## Core Responsibilities
- Defines a static dispatch table (`commands[]`) mapping command name strings to handler functions
- Implements `CG_ConsoleCommand` to look up and invoke handlers when the engine forwards an unrecognized command to cgame
- Implements `CG_InitConsoleCommands` to register all commands with the engine for tab-completion
- Handles scoreboard show/hide state and optional score refresh requests
- Provides tell/voice-tell shortcuts targeting crosshair player or last attacker
- Under `MISSIONPACK`: handles HUD reloading, team orders, scoreboard scrolling, and SP win/lose sequences

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `consoleCommand_t` | struct | Pairs a command name string with a void function pointer for dispatch |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `commands[]` | `static consoleCommand_t[]` | file-static | Master dispatch table of all cgame-handled console commands |
| `menuScoreboard` | `menuDef_t *` (extern, MISSIONPACK) | global (defined elsewhere) | Pointer to the scoreboard menu used for feeder scrolling |

## Key Functions / Methods

### CG_ConsoleCommand
- **Signature:** `qboolean CG_ConsoleCommand( void )`
- **Purpose:** Engine-facing entry point; called when the client system has a command that wasn't handled internally. Performs a linear search through `commands[]` using case-insensitive comparison.
- **Inputs:** Command token 0 read via `CG_Argv(0)`
- **Outputs/Return:** `qtrue` if handled, `qfalse` to let the engine forward the command to the server
- **Side effects:** Calls the matched handler function, which may mutate `cg`/`cgs` state or issue trap calls
- **Calls:** `CG_Argv`, `Q_stricmp`, handler functions in `commands[]`
- **Notes:** Linear O(n) search; command table size ~40 entries

### CG_InitConsoleCommands
- **Signature:** `void CG_InitConsoleCommands( void )`
- **Purpose:** Registers all cgame and server-forwarded commands with the engine so tab-completion works.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Issues `trap_AddCommand` for every entry in `commands[]`, plus ~25 server-side command names (kill, say, team, vote, etc.)
- **Calls:** `trap_AddCommand`
- **Notes:** `loaddefered` is registered with a known spelling error preserved for demo compatibility

### CG_ScoresDown_f
- **Signature:** `static void CG_ScoresDown_f( void )`
- **Purpose:** Handles `+scores`; requests fresh score data from the server if cached data is older than 2 seconds, otherwise shows cached scores immediately.
- **Inputs:** None (reads `cg.scoresRequestTime`, `cg.time`)
- **Outputs/Return:** void
- **Side effects:** Sets `cg.showScores`, `cg.numScores`, `cg.scoresRequestTime`; sends `"score"` client command; under MISSIONPACK calls `CG_BuildSpectatorString`
- **Calls:** `trap_SendClientCommand`, `CG_BuildSpectatorString` (MISSIONPACK)

### CG_ScoresUp_f
- **Signature:** `static void CG_ScoresUp_f( void )`
- **Purpose:** Handles `-scores`; hides the scoreboard and records the fade-out start time.
- **Side effects:** Sets `cg.showScores = qfalse`, `cg.scoreFadeTime = cg.time`

### CG_TellTarget_f / CG_TellAttacker_f
- **Signature:** `static void CG_TellTarget_f( void )` / `static void CG_TellAttacker_f( void )`
- **Purpose:** Sends a `tell <clientNum> <message>` command to the server targeting the player under the crosshair or the last attacker respectively.
- **Side effects:** `trap_SendClientCommand`
- **Calls:** `CG_CrosshairPlayer` or `CG_LastAttacker`, `trap_Args`, `Com_sprintf`, `trap_SendClientCommand`

### CG_LoadHud_f *(MISSIONPACK only)*
- **Signature:** `static void CG_LoadHud_f( void )`
- **Purpose:** Reloads HUD menu definitions at runtime from the path in `cg_hudFiles` cvar (defaults to `"ui/hud.txt"`).
- **Side effects:** Calls `String_Init`, `Menu_Reset`, `CG_LoadMenus`; resets `menuScoreboard` to NULL

### CG_NextOrder_f *(MISSIONPACK only)*
- **Signature:** `static void CG_NextOrder_f( void )`
- **Purpose:** Cycles through team task orders (OFFENSE→…→CAMP→OFFENSE), skipping RETRIEVE/ESCORT if flag conditions aren't met.
- **Side effects:** Mutates `cgs.currentOrder`, `cgs.orderPending`, `cgs.orderTime`
- **Notes:** Enforces team-leader-only restriction; assumes `TEAMTASK_OFFENSE == 0`, `TEAMTASK_CAMP` is last

### CG_StartOrbit_f
- **Signature:** `static void CG_StartOrbit_f( void )`
- **Purpose:** Developer-only toggle for third-person camera orbit mode; no-ops if `developer` cvar is 0.
- **Side effects:** Sets `cg_cameraOrbit`, `cg_thirdPerson`, `cg_thirdPersonAngle`, `cg_thirdPersonRange` cvars

- **Notes on trivial helpers:** `CG_SizeUp_f`/`CG_SizeDown_f` adjust `cg_viewsize` ±10. `CG_Viewpos_f` prints current view position. Voice-tell, task, and taunt functions are thin wrappers that format and send `vsay`/`vtell`/`teamtask` console or client commands.

## Control Flow Notes
- Called during the **input/command phase** each frame: `CG_ConsoleCommand` is invoked by the engine when it has a command token not handled at the system level.
- `CG_InitConsoleCommands` is called once during **cgame initialization** (from `CG_Init`).
- No per-frame update logic exists here; all functions are purely event-driven (command invocation).

## External Dependencies
- `cg_local.h` — full cgame state (`cg_t cg`, `cgs_t cgs`), trap declarations, and function prototypes
- `../ui/ui_shared.h` — `menuDef_t`, `Menu_ScrollFeeder`, `String_Init`, `Menu_Reset`
- **Defined elsewhere:** `CG_CrosshairPlayer`, `CG_LastAttacker`, `CG_LoadMenus`, `CG_AddBufferedSound`, `CG_CenterPrint`, `CG_BuildSpectatorString`, `CG_SelectNextPlayer`/`CG_SelectPrevPlayer`, `CG_OtherTeamHasFlag`, `CG_YourTeamHasFlag`, `CG_LoadDeferredPlayers`, all `CG_TestModel_*`/`CG_Zoom*`/`CG_*Weapon_f` functions, all `trap_*` syscalls
