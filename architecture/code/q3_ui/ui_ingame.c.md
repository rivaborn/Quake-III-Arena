# code/q3_ui/ui_ingame.c

## File Purpose
Implements the in-game pause menu for Quake III Arena, presenting a vertical list of text buttons that allow the player to access game management options (team, bots, setup, server info, restart, quit, resume, leave) while paused mid-session.

## Core Responsibilities
- Define and initialize all menu items (`ingamemenu_t`) for the in-game overlay menu
- Conditionally gray out menu items based on runtime cvars (e.g., `sv_running`, `bot_enable`, `g_gametype`)
- Dispatch UI navigation events to the appropriate sub-menu or game command via `InGame_Event`
- Pre-cache the frame background shader via `InGame_Cache`
- Reset menu stack to top-level and push the initialized menu via `UI_InGameMenu`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `ingamemenu_t` | struct (typedef) | Aggregates the `menuframework_s` and all `menubitmap_s`/`menutext_s` items for the in-game menu |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_ingame` | `ingamemenu_t` | static (file) | Sole instance of the in-game menu; persists across the draw/key loop |

## Key Functions / Methods

### InGame_RestartAction
- **Signature:** `static void InGame_RestartAction( qboolean result )`
- **Purpose:** Confirmation callback for "RESTART ARENA"; executes `map_restart 0` if confirmed.
- **Inputs:** `result` — qtrue if user confirmed, qfalse to cancel.
- **Outputs/Return:** void
- **Side effects:** Pops the menu stack; sends `map_restart 0` to the command buffer.
- **Calls:** `UI_PopMenu`, `trap_Cmd_ExecuteText`
- **Notes:** Early-returns with no action on `qfalse`.

---

### InGame_QuitAction
- **Signature:** `static void InGame_QuitAction( qboolean result )`
- **Purpose:** Confirmation callback for "EXIT GAME"; dismisses the menu and navigates to the credits screen.
- **Inputs:** `result` — qtrue if confirmed.
- **Outputs/Return:** void
- **Side effects:** Pops menu stack, pushes credits menu.
- **Calls:** `UI_PopMenu`, `UI_CreditMenu`

---

### InGame_Event
- **Signature:** `void InGame_Event( void *ptr, int notification )`
- **Purpose:** Unified callback for all menu items; dispatches on item `id` to the appropriate sub-menu or command.
- **Inputs:** `ptr` — pointer to the `menucommon_s` that triggered the event; `notification` — event type (only `QM_ACTIVATED` is handled).
- **Outputs/Return:** void
- **Side effects:** May push a new menu, issue a console command (`disconnect`), or open a confirm dialog.
- **Calls:** `UI_TeamMainMenu`, `UI_SetupMenu`, `trap_Cmd_ExecuteText`, `UI_ConfirmMenu`, `UI_ServerInfoMenu`, `UI_AddBotsMenu`, `UI_RemoveBotsMenu`, `UI_TeamOrdersMenu`, `UI_PopMenu`

---

### InGame_MenuInit
- **Signature:** `void InGame_MenuInit( void )`
- **Purpose:** Zeroes `s_ingame`, sets per-item properties (position, label, color, callback, id), applies conditional graying, then registers all items with `Menu_AddItem`.
- **Inputs:** None (reads cvars and client state internally).
- **Outputs/Return:** void
- **Side effects:** Mutates global `s_ingame`; reads `sv_running`, `bot_enable`, `g_gametype`, client config string for team membership.
- **Calls:** `InGame_Cache`, `trap_Cvar_VariableValue`, `trap_GetClientState`, `trap_GetConfigString`, `Info_ValueForKey`, `Menu_AddItem`
- **Notes:** "TEAM ORDERS" is grayed for non-team gametypes and for spectators. "ADD BOTS" / "REMOVE BOTS" require `sv_running && bot_enable && !GT_SINGLE_PLAYER`. "RESTART ARENA" requires `sv_running`.

---

### InGame_Cache
- **Signature:** `void InGame_Cache( void )`
- **Purpose:** Pre-registers the frame background shader with the renderer.
- **Inputs:** None.
- **Outputs/Return:** void
- **Side effects:** Registers `INGAME_FRAME` shader (renderer-side alloc).
- **Calls:** `trap_R_RegisterShaderNoMip`

---

### UI_InGameMenu
- **Signature:** `void UI_InGameMenu( void )`
- **Purpose:** Public entry point; resets the menu stack, positions the cursor, initializes and pushes the in-game menu.
- **Inputs:** None.
- **Outputs/Return:** void
- **Side effects:** Writes `uis.menusp`, `uis.cursorx`, `uis.cursory`; pushes onto the menu stack.
- **Calls:** `InGame_MenuInit`, `UI_PushMenu`
- **Notes:** Forces `uis.menusp = 0` to clear any previously stacked menus, making this a top-level overlay.

## Control Flow Notes
Activated when the player presses Escape in-game. `UI_InGameMenu` is the engine-facing entry point; it initializes the menu fresh each time (re-evaluating cvar-driven graying). Rendering and input are handled by the shared `menuframework_s` system (`Menu_Draw` / `Menu_DefaultKey`) — this file contributes no frame/render loop code of its own.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, menu framework types)
- **Defined elsewhere:** `UI_PopMenu`, `UI_PushMenu`, `UI_ConfirmMenu`, `UI_CreditMenu`, `UI_TeamMainMenu`, `UI_SetupMenu`, `UI_ServerInfoMenu`, `UI_AddBotsMenu`, `UI_RemoveBotsMenu`, `UI_TeamOrdersMenu`, `Menu_AddItem`, `trap_*` syscall wrappers, `uis` global, `color_red`, `Info_ValueForKey`
