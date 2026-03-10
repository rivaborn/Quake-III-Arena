# code/q3_ui/ui_teamorders.c

## File Purpose
Implements the in-game Team Orders menu for Quake III Arena, allowing players to issue commands to bot teammates. It presents a two-step selection UI: first choose a bot (or "Everyone"), then choose an order, which is transmitted as a `say_team` chat message.

## Core Responsibilities
- Build a dynamic list of bot teammates from server config strings
- Render a scrollable, owner-drawn proportional-font list widget
- Handle two-phase selection: bot target → order message
- Format and dispatch `say_team` commands with the selected bot name interpolated
- Guard menu access (team game only, non-spectators)
- Pre-cache required artwork shaders

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `teamOrdersMenuInfo_t` | struct | All menu state: framework, widgets, gametype, bot name array, selection index |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `teamOrdersMenuInfo` | `teamOrdersMenuInfo_t` | static (file) | Single instance holding all menu state |
| `ctfOrders` | `const char *[]` | static (file) | Display strings for CTF order choices |
| `ctfMessages` | `const char *[]` | static (file) | `say_team` format strings for CTF orders |
| `teamOrders` | `const char *[]` | static (file) | Display strings for team deathmatch order choices |
| `teamMessages` | `const char *[]` | static (file) | `say_team` format strings for team DM orders |

## Key Functions / Methods

### UI_TeamOrdersMenu_BuildBotList
- **Signature:** `static void UI_TeamOrdersMenu_BuildBotList( void )`
- **Purpose:** Populates `teamOrdersMenuInfo.bots[]` with names of bot teammates by iterating server player slots.
- **Inputs:** None (reads server config strings via traps)
- **Outputs/Return:** Void; writes into `teamOrdersMenuInfo.botNames`, `numBots`, `gametype`
- **Side effects:** Calls `trap_GetClientState`, `trap_GetConfigString`; mutates global menu state
- **Calls:** `trap_GetClientState`, `trap_GetConfigString`, `Info_ValueForKey`, `atoi`, `Q_strncpyz`, `Q_CleanStr`
- **Notes:** Skips the local client's own slot; skips non-bots (zero `skill`); skips players on other teams. `playerTeam` is initialized to `TEAM_SPECTATOR` to suppress a potential uninitialized-use warning, but is only validly set when `n == cs.clientNum` — then `continue` is hit immediately, so the team comparison against `botTeam` always uses a stale/default value. This is a latent bug.

### UI_TeamOrdersMenu_SetList
- **Signature:** `static void UI_TeamOrdersMenu_SetList( int id )`
- **Purpose:** Switches the single shared `menulist_s` widget between the bot list, CTF order list, and team order list by updating `itemnames`, `numitems`, and recalculating `bottom`.
- **Inputs:** `id` — one of `ID_LIST_BOTS`, `ID_LIST_CTF_ORDERS`, `ID_LIST_TEAM_ORDERS`
- **Outputs/Return:** Void; mutates `teamOrdersMenuInfo.list`
- **Side effects:** Modifies widget bounding box (`bottom`)
- **Calls:** None
- **Notes:** `default` falls through to `ID_LIST_BOTS`.

### UI_TeamOrdersMenu_ListEvent
- **Signature:** `static void UI_TeamOrdersMenu_ListEvent( void *ptr, int event )`
- **Purpose:** Callback for list activation. First activation selects the bot and transitions list to orders; second activation formats and dispatches the `say_team` command.
- **Inputs:** `ptr` — `menulist_s *`; `event` — menu event type
- **Outputs/Return:** Void
- **Side effects:** Calls `trap_Cmd_ExecuteText` to issue `say_team`; calls `UI_PopMenu`
- **Calls:** `Com_sprintf`, `va`, `trap_Cmd_ExecuteText`, `UI_PopMenu`, `UI_TeamOrdersMenu_SetList`
- **Notes:** Uses `%s` interpolation from `ctfMessages`/`teamMessages` with the stored bot name.

### UI_TeamOrdersMenu_Key
- **Signature:** `sfxHandle_t UI_TeamOrdersMenu_Key( int key )`
- **Purpose:** Custom key handler for the menu; provides mouse-click list hit-testing and arrow-key navigation on the active list widget.
- **Inputs:** `key` — key code
- **Outputs/Return:** `sfxHandle_t` (sound to play)
- **Side effects:** Modifies `list.curvalue`/`oldvalue`; invokes list callback on click
- **Calls:** `Menu_ItemAtCursor`, `UI_CursorInRect`, `Menu_DefaultKey`
- **Notes:** Mouse click index is computed from `uis.cursory` relative to list top.

### UI_TeamOrdersMenu_ListDraw
- **Signature:** `static void UI_TeamOrdersMenu_ListDraw( void *self )`
- **Purpose:** Owner-draw callback rendering list items as proportional strings; highlights the selected item in yellow (pulsing if focused), others in orange.
- **Inputs:** `self` — `menulist_s *`
- **Outputs/Return:** Void
- **Side effects:** Issues render calls via `UI_DrawProportionalString`
- **Calls:** `UI_DrawProportionalString`
- **Notes:** Hardcodes `x = 320` (screen center), ignoring `generic.x`.

### UI_TeamOrdersMenu_Init
- **Signature:** `static void UI_TeamOrdersMenu_Init( void )`
- **Purpose:** Initializes all menu widgets (banner, frame, list, back button) and adds them to the framework.
- **Side effects:** Calls `UI_TeamOrdersMenu_Cache`, `memset`, `UI_TeamOrdersMenu_BuildBotList`, `Menu_AddItem`, `UI_TeamOrdersMenu_SetList`

### UI_TeamOrdersMenu_Cache
- **Signature:** `void UI_TeamOrdersMenu_Cache( void )`
- **Purpose:** Pre-registers all artwork shaders needed by this menu.
- **Calls:** `trap_R_RegisterShaderNoMip` (×3)

### UI_TeamOrdersMenu / UI_TeamOrdersMenu_f
- **Signature:** `void UI_TeamOrdersMenu( void )` / `void UI_TeamOrdersMenu_f( void )`
- **Purpose:** Public entry points. `UI_TeamOrdersMenu` initializes and pushes the menu. `UI_TeamOrdersMenu_f` is the console-command handler that validates game type (`>= GT_TEAM`) and player team (`!= TEAM_SPECTATOR`) before calling `UI_TeamOrdersMenu`.
- **Calls:** `UI_TeamOrdersMenu_Init`, `UI_PushMenu`, `trap_GetConfigString`, `trap_GetClientState`, `Info_ValueForKey`, `atoi`

## Control Flow Notes
Called from the in-game menu or console command binding. `UI_TeamOrdersMenu_f` is the guarded entry. On activation, `Init` builds the bot list and shows it; selecting a bot transitions the list to orders; selecting an order dispatches the chat command and pops the menu. No per-frame update logic.

## External Dependencies
- **Includes:** `ui_local.h` → pulls in `q_shared.h`, `bg_public.h`, UI framework types and trap declarations
- **Defined elsewhere:** `trap_GetClientState`, `trap_GetConfigString`, `trap_Cmd_ExecuteText`, `trap_R_RegisterShaderNoMip`, `UI_PushMenu`, `UI_PopMenu`, `Menu_AddItem`, `Menu_ItemAtCursor`, `Menu_DefaultKey`, `UI_DrawProportionalString`, `UI_CursorInRect`, `Com_sprintf`, `va`, `Q_strncpyz`, `Q_CleanStr`, `Info_ValueForKey`, `uis` (global UI state), game constants `GT_CTF`, `GT_TEAM`, `TEAM_SPECTATOR`, `CS_SERVERINFO`, `CS_PLAYERS`
