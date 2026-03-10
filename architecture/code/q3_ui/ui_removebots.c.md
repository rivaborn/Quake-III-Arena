# code/q3_ui/ui_removebots.c

## File Purpose
Implements the in-game "Remove Bots" menu for Quake III Arena's legacy UI module. It allows a human player to view currently connected bot clients and kick one by client number via a console command.

## Core Responsibilities
- Enumerate active bot clients from server config strings by checking for a non-zero `skill` field
- Display up to 7 bot names in a scrollable list
- Track which bot entry is selected and visually distinguish it (orange vs. white color)
- Issue a `clientkick <num>` command when the user activates the Delete button
- Register and cache all required artwork shaders on demand
- Push/pop the menu onto the UI menu stack

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `removeBotsMenuInfo_t` | struct | Aggregates all menu widgets, bot name buffers, bot client numbers, scroll/selection state, and bot count into a single menu descriptor |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `removeBotsMenuInfo` | `removeBotsMenuInfo_t` | static (file) | Sole instance of the remove-bots menu; re-initialized each time the menu is opened |

## Key Functions / Methods

### UI_RemoveBotsMenu_GetBots
- **Signature:** `static void UI_RemoveBotsMenu_GetBots( void )`
- **Purpose:** Scans all player slots up to `sv_maxclients`, identifies bots by a non-zero `skill` config key, and populates `botClientNums[]` and `numBots`.
- **Inputs:** None (reads from server config strings via `trap_GetConfigString`).
- **Outputs/Return:** void; writes `removeBotsMenuInfo.botClientNums[]` and `removeBotsMenuInfo.numBots`.
- **Side effects:** Two `trap_GetConfigString` calls (`CS_SERVERINFO`, `CS_PLAYERS + n`).
- **Calls:** `trap_GetConfigString`, `Info_ValueForKey`, `atoi`.
- **Notes:** Bots are identified solely by `skill != 0`; human players have `skill == 0`.

### UI_RemoveBotsMenu_SetBotNames
- **Signature:** `static void UI_RemoveBotsMenu_SetBotNames( void )`
- **Purpose:** Fills the display name buffers for up to 7 visible bot slots starting at `baseBotNum`, stripping color codes with `Q_CleanStr`.
- **Inputs:** None (reads `botClientNums`, `baseBotNum`, `numBots`).
- **Outputs/Return:** void; writes `removeBotsMenuInfo.botnames[0..6]`.
- **Side effects:** One `trap_GetConfigString` call per visible bot.
- **Calls:** `trap_GetConfigString`, `Info_ValueForKey`, `Q_strncpyz`, `Q_CleanStr`.
- **Notes:** Called both during init and on every scroll event.

### UI_RemoveBotsMenu_DeleteEvent
- **Signature:** `static void UI_RemoveBotsMenu_DeleteEvent( void* ptr, int event )`
- **Purpose:** Callback for the Delete button; kicks the currently selected bot via `clientkick`.
- **Inputs:** `ptr` — unused widget pointer; `event` — must be `QM_ACTIVATED`.
- **Outputs/Return:** void.
- **Side effects:** Calls `trap_Cmd_ExecuteText(EXEC_APPEND, "clientkick <n>\n")`, which is handled by the server.
- **Calls:** `trap_Cmd_ExecuteText`, `va`.
- **Notes:** Uses `baseBotNum + selectedBotNum` to index into `botClientNums`.

### UI_RemoveBotsMenu_BotEvent
- **Signature:** `static void UI_RemoveBotsMenu_BotEvent( void* ptr, int event )`
- **Purpose:** Callback for individual bot name items; updates the selection highlight.
- **Inputs:** `ptr` — `menucommon_s*` cast; `id` field used to derive new index.
- **Outputs/Return:** void.
- **Side effects:** Sets previous selection to `color_orange`, new selection to `color_white`.
- **Calls:** None beyond field access.

### UI_RemoveBots_Cache
- **Signature:** `void UI_RemoveBots_Cache( void )`
- **Purpose:** Pre-registers menu artwork shaders so they are available before first draw.
- **Inputs/Outputs:** None.
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` for 5 art paths.

### UI_RemoveBotsMenu_Init
- **Signature:** `static void UI_RemoveBotsMenu_Init( void )`
- **Purpose:** Zeroes the menu state, populates widget descriptors, registers items with the menu framework, and sets initial selection/scroll positions.
- **Calls:** `UI_RemoveBots_Cache`, `UI_RemoveBotsMenu_GetBots`, `UI_RemoveBotsMenu_SetBotNames`, `Menu_AddItem`, `memset`.

### UI_RemoveBotsMenu
- **Signature:** `void UI_RemoveBotsMenu( void )`
- **Purpose:** Public entry point; initializes and pushes the menu.
- **Calls:** `UI_RemoveBotsMenu_Init`, `UI_PushMenu`.

- **Notes:** `UI_RemoveBotsMenu_UpEvent` / `UI_RemoveBotsMenu_DownEvent` scroll `baseBotNum` by ±1 and refresh names; `UI_RemoveBotsMenu_BackEvent` calls `UI_PopMenu`.

## Control Flow Notes
This file is UI-only and has no per-frame update hook. It is entered via `UI_RemoveBotsMenu()` (push), driven entirely by menu framework callbacks on user input events, and exits via `UI_PopMenu`. All bot enumeration happens once at open time (`Init`); the list is refreshed on scroll but not polled live.

## External Dependencies
- `ui_local.h` — all menu framework types, widget types, trap syscall declarations, color vectors
- `trap_GetConfigString` — reads `CS_SERVERINFO` and `CS_PLAYERS + n` (defined in engine/syscall layer)
- `trap_Cmd_ExecuteText` — issues console commands to the engine
- `trap_R_RegisterShaderNoMip` — registers 2D art assets
- `Info_ValueForKey`, `Q_strncpyz`, `Q_CleanStr` — defined in `q_shared.c`
- `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem` — defined in `ui_atoms.c` / `ui_qmenu.c`
- `MAX_BOTS` — defined in `bg_public.h`
