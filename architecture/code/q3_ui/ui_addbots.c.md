# code/q3_ui/ui_addbots.c

## File Purpose
Implements the in-game "Add Bots" menu for Quake III Arena, allowing players to add AI bots to a running server session. It builds a scrollable list of available bots with skill level and team selection controls.

## Core Responsibilities
- Initialize and display the Add Bots menu UI with all interactive widgets
- Retrieve and alphabetically sort available bot names from the game's bot info database
- Scroll a paginated list of up to 7 bot names at a time
- Handle bot selection highlighting and dispatch the `addbot` server command on confirmation
- Pre-cache all menu art assets for rendering
- Adapt team options based on the current game type (FFA vs. team modes)

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `addBotsMenuInfo_t` | struct | Aggregates all menu widgets, bot list state, scroll position, delay counter, and bot name buffers |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `addBotsMenuInfo` | `addBotsMenuInfo_t` | static (file) | Singleton holding all runtime state for the Add Bots menu |
| `skillNames` | `const char*[]` | static (file) | String table for the 5 skill level entries in the spin control |
| `teamNames1` | `const char*[]` | static (file) | Team options for FFA modes ("Free") |
| `teamNames2` | `const char*[]` | static (file) | Team options for team modes ("Red", "Blue") |

## Key Functions / Methods

### UI_AddBotsMenu_FightEvent
- **Signature:** `static void UI_AddBotsMenu_FightEvent(void *ptr, int event)`
- **Purpose:** Callback for the "Accept/Fight" button; issues the `addbot` console command.
- **Inputs:** Widget pointer (unused beyond event check), `event` code.
- **Outputs/Return:** None.
- **Side effects:** Appends `addbot <name> <skill> <team> <delay>` to the command buffer; increments `addBotsMenuInfo.delay` by 1500 ms.
- **Calls:** `trap_Cmd_ExecuteText`, `va`
- **Notes:** The delay staggers multiple successive bot additions to avoid spawn collisions.

### UI_AddBotsMenu_BotEvent
- **Signature:** `static void UI_AddBotsMenu_BotEvent(void *ptr, int event)`
- **Purpose:** Callback for a bot name text item; updates which bot is selected by toggling colors.
- **Inputs:** Widget pointer (carries `id` encoding the list index), `event`.
- **Outputs/Return:** None.
- **Side effects:** Sets previously selected bot text to `color_orange`, new selection to `color_white`.
- **Calls:** None (reads `((menucommon_s*)ptr)->id`).

### UI_AddBotsMenu_SetBotNames
- **Signature:** `static void UI_AddBotsMenu_SetBotNames(void)`
- **Purpose:** Refreshes the 7 visible bot name strings from the sorted index array starting at `baseBotNum`.
- **Inputs:** None (reads global `addBotsMenuInfo`).
- **Outputs/Return:** None.
- **Side effects:** Writes into `addBotsMenuInfo.botnames[0..6]`.
- **Calls:** `UI_GetBotInfoByNumber`, `Info_ValueForKey`, `Q_strncpyz`

### UI_AddBotsMenu_SortCompare / UI_AddBotsMenu_GetSortedBotNums
- **Purpose:** Build and sort `sortedBotNums[]` alphabetically by bot name using `qsort`.
- **Calls:** `UI_GetBotInfoByNumber`, `Info_ValueForKey`, `Q_stricmp`, `qsort`
- **Notes:** Comparator uses `QDECL` calling convention for portability with the CRT `qsort`.

### UI_AddBotsMenu_Draw
- **Signature:** `static void UI_AddBotsMenu_Draw(void)`
- **Purpose:** Custom draw callback assigned to the menu; renders banner text, background art, and all menu items.
- **Calls:** `UI_DrawBannerString`, `UI_DrawNamedPic`, `Menu_Draw`

### UI_AddBotsMenu_Init
- **Signature:** `static void UI_AddBotsMenu_Init(void)`
- **Purpose:** Allocates and wires up all menu widgets, reads server info for game type, and populates the bot list.
- **Inputs:** None.
- **Outputs/Return:** None.
- **Side effects:** Zeroes and populates `addBotsMenuInfo`; calls `UI_AddBots_Cache`; registers all items with `Menu_AddItem`.
- **Calls:** `trap_GetConfigString`, `UI_AddBots_Cache`, `UI_GetNumBots`, `UI_AddBotsMenu_GetSortedBotNums`, `UI_AddBotsMenu_SetBotNames`, `Menu_AddItem`, `trap_Cvar_VariableValue`, `Com_Clamp`

### UI_AddBots_Cache
- **Signature:** `void UI_AddBots_Cache(void)`
- **Purpose:** Pre-registers all shader/bitmap assets used by this menu with the renderer.
- **Calls:** `trap_R_RegisterShaderNoMip` (×8)

### UI_AddBotsMenu
- **Signature:** `void UI_AddBotsMenu(void)`
- **Purpose:** Public entry point; initializes the menu and pushes it onto the UI menu stack.
- **Calls:** `UI_AddBotsMenu_Init`, `UI_PushMenu`

## Control Flow Notes
This file is UI-layer only — it has no presence in the main game loop frame. `UI_AddBotsMenu` is called from elsewhere in the UI (e.g., the in-game menu) to push the screen. The menu's `draw` callback (`UI_AddBotsMenu_Draw`) is invoked each frame by the generic `UI_Refresh` path while the menu is active. Bot addition is deferred entirely to the server command buffer via `EXEC_APPEND`.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types, trap syscalls)
- **Defined elsewhere:**
  - `UI_GetBotInfoByNumber`, `UI_GetNumBots` — `ui_gameinfo.c`
  - `Menu_Draw`, `Menu_AddItem`, `Menu_AddItem` — `ui_qmenu.c`
  - `UI_PushMenu`, `UI_PopMenu`, `UI_DrawBannerString`, `UI_DrawNamedPic` — `ui_atoms.c`
  - `trap_*` syscall wrappers — `ui_syscalls.c`
  - `Info_ValueForKey`, `Q_strncpyz`, `Q_stricmp`, `Com_Clamp` — `q_shared.c`
