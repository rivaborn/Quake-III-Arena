# code/q3_ui/ui_team.c

## File Purpose
Implements the in-game Team Selection overlay menu for Quake III Arena, allowing players to join the red team, blue team, free-for-all, or spectate. Menu items are conditionally grayed out based on the current server game type.

## Core Responsibilities
- Define and initialize the team selection menu (`s_teammain`)
- Register the decorative frame shader asset via cache call
- Handle menu item activation events by sending server commands (`cmd team red/blue/free/spectator`)
- Query `CS_SERVERINFO` to determine current game type and disable irrelevant options
- Push the initialized menu onto the UI menu stack

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `teammain_t` | struct | Aggregates the menu framework and all five menu items (frame bitmap + four text buttons) into a single cohesive menu object |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_teammain` | `static teammain_t` | static (file) | Sole instance of the team selection menu; zeroed on each `TeamMain_MenuInit` call |

## Key Functions / Methods

### TeamMain_MenuEvent
- **Signature:** `static void TeamMain_MenuEvent( void* ptr, int event )`
- **Purpose:** Callback invoked by the menu system when a button is activated; dispatches the appropriate server-side `cmd team` command and closes the menu.
- **Inputs:** `ptr` — pointer to the activating `menucommon_s`; `event` — notification type (only `QM_ACTIVATED` is acted upon)
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_Cmd_ExecuteText(EXEC_APPEND, ...)` to enqueue a team-change command; calls `UI_ForceMenuOff()` to close all menus immediately.
- **Calls:** `trap_Cmd_ExecuteText`, `UI_ForceMenuOff`
- **Notes:** Returns early for any event other than `QM_ACTIVATED`; four IDs handled: `ID_JOINRED`, `ID_JOINBLUE`, `ID_JOINGAME`, `ID_SPECTATE`.

### TeamMain_MenuInit
- **Signature:** `void TeamMain_MenuInit( void )`
- **Purpose:** Zeros and configures the entire `s_teammain` menu: layout, widget properties, and per-gametype graying logic.
- **Inputs:** None (reads `CS_SERVERINFO` cvar internally)
- **Outputs/Return:** `void`
- **Side effects:** Writes to `s_teammain` global; calls `TeamMain_Cache()`; calls `trap_GetConfigString` to query server info; calls `Menu_AddItem` five times to register widgets.
- **Calls:** `TeamMain_Cache`, `trap_GetConfigString`, `Info_ValueForKey`, `atoi`, `Menu_AddItem`
- **Notes:** `joinred`/`joinblue` are grayed for `GT_SINGLE_PLAYER`, `GT_FFA`, `GT_TOURNAMENT`; `joingame` is grayed for `GT_TEAM`/`GT_CTF`. All text items share `colorRed` despite their differing roles — likely a copy-paste oversight for `joinblue`.

### TeamMain_Cache
- **Signature:** `void TeamMain_Cache( void )`
- **Purpose:** Pre-registers the frame bitmap shader with the renderer so it is resident when the menu draws.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` with `TEAMMAIN_FRAME` (`"menu/art/cut_frame"`).
- **Calls:** `trap_R_RegisterShaderNoMip`
- **Notes:** Called from `TeamMain_MenuInit`; also exposed for external pre-caching.

### UI_TeamMainMenu
- **Signature:** `void UI_TeamMainMenu( void )`
- **Purpose:** Public entry point that initializes and displays the team selection menu.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `TeamMain_MenuInit`, then `UI_PushMenu` to make the menu active.
- **Calls:** `TeamMain_MenuInit`, `UI_PushMenu`
- **Notes:** This is the sole externally visible entry point declared in `ui_local.h`.

## Control Flow Notes
This file has no frame/update loop participation. It is activated on-demand: `UI_TeamMainMenu` is called from elsewhere (likely `ui_ingame.c`) in response to a player selecting "Team" from the in-game menu. The menu persists until the player selects an option, at which point `UI_ForceMenuOff` terminates it entirely.

## External Dependencies
- `ui_local.h` — pulls in `menuframework_s`, `menubitmap_s`, `menutext_s`, `menucommon_s`, `QM_ACTIVATED`, `QMF_*` flags, `MTYPE_*` constants, game type enums (`GT_TEAM`, `GT_CTF`, etc.), `CS_SERVERINFO`, and all `trap_*` / `UI_*` function declarations
- `trap_Cmd_ExecuteText` — defined in `ui_syscalls.c`, bridges to engine
- `trap_GetConfigString` — defined in `ui_syscalls.c`, bridges to engine
- `trap_R_RegisterShaderNoMip` — defined in `ui_syscalls.c`, bridges to renderer
- `Info_ValueForKey` — defined in `q_shared.c`
- `UI_ForceMenuOff`, `UI_PushMenu`, `Menu_AddItem` — defined in `ui_atoms.c` / `ui_qmenu.c`
