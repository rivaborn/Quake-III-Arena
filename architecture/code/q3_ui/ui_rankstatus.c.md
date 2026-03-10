# code/q3_ui/ui_rankstatus.c

## File Purpose
Implements a modal status dialog for the GRank (Global Ranking) online ranking system, displaying error or result messages when a ranking operation completes. It maps `grank_status_t` codes to human-readable strings and routes the user to appropriate follow-up menus on dismissal.

## Core Responsibilities
- Read `client_status` cvar to determine the current `grank_status_t` code
- Map ranking status codes to display strings (e.g., "Invalid password", "Timed out")
- Build and display a simple two-item menu: a static message and an OK button
- On OK, pop this menu and push the appropriate follow-up menu (rankings, login, signup) based on the original status code
- Early-exit silently for benign statuses (`QGR_STATUS_NEW`, `QGR_STATUS_PENDING`, `QGR_STATUS_SPECTATOR`, `QGR_STATUS_ACTIVE`)
- Pre-cache the frame shader via `RankStatus_Cache`

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `rankstatus_t` | struct | Aggregates the menu framework, frame bitmap, message text, and OK button for this dialog |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_rankstatus` | `rankstatus_t` | static | Sole instance of the rank status menu layout |
| `s_rankstatus_menu` | `menuframework_s` | static | Declared but unused; superseded by `s_rankstatus.menu` |
| `s_rankstatus_ok` | `menuaction_s` | static | Declared but unused; superseded by `s_rankstatus.ok` |
| `s_status` | `grank_status_t` | static | Captures the status code at menu open time; drives OK callback routing |
| `s_rankstatus_message` | `char*` | static | Points to a string literal selected from the status switch; assigned to the message widget |
| `s_rankingstatus_color_prompt` | `vec4_t` | static | Orange RGBA color `{1.00, 0.43, 0.00, 1.00}` for the message text |

## Key Functions / Methods

### RankStatus_MenuEvent
- **Signature:** `static void RankStatus_MenuEvent( void* ptr, int event )`
- **Purpose:** Callback for all menu item events; only handles `QM_ACTIVATED` on `ID_OK`. Pops this menu then pushes the contextually appropriate ranking sub-menu.
- **Inputs:** `ptr` — `menucommon_s*` with `.id`; `event` — notification type
- **Outputs/Return:** `void`
- **Side effects:** Calls `UI_PopMenu()` and one or more of `UI_RankingsMenu()`, `UI_LoginMenu()`, `UI_SignupMenu()`
- **Calls:** `UI_PopMenu`, `UI_RankingsMenu`, `UI_LoginMenu`, `UI_SignupMenu`
- **Notes:** `QGR_STATUS_NO_USER`, `QGR_STATUS_NO_MEMBERSHIP`, `QGR_STATUS_TIMEOUT`, `QGR_STATUS_INVALIDUSER`, and `QGR_STATUS_ERROR` all route identically to `UI_RankingsMenu` only; the distinction is preserved for future differentiation.

### RankStatus_MenuInit
- **Signature:** `void RankStatus_MenuInit( void )`
- **Purpose:** Zeroes `s_rankstatus`, calls `RankStatus_Cache`, configures all widget fields with hardcoded pixel positions, then registers them with the menu framework.
- **Inputs:** None (reads `s_rankstatus_message` global)
- **Outputs/Return:** `void`
- **Side effects:** Writes `s_rankstatus`; calls `trap_R_RegisterShaderNoMip` indirectly via `RankStatus_Cache`; calls `Menu_AddItem` three times
- **Calls:** `RankStatus_Cache`, `Menu_AddItem`
- **Notes:** Layout uses absolute 640×480 virtual-screen coordinates. The `ok` item uses `QMF_PULSEIFFOCUS` for visual feedback; the `message` item is `QMF_INACTIVE`.

### RankStatus_Cache
- **Signature:** `void RankStatus_Cache( void )`
- **Purpose:** Pre-registers the frame background shader with the renderer to avoid a hitch on first display.
- **Inputs:** None
- **Outputs/Return:** `void`
- **Side effects:** Calls `trap_R_RegisterShaderNoMip( "menu/art/cut_frame" )`
- **Calls:** `trap_R_RegisterShaderNoMip`

### UI_RankStatusMenu
- **Signature:** `void UI_RankStatusMenu( void )`
- **Purpose:** Primary entry point. Reads `client_status` cvar, selects a message string, initialises the menu, resets rank user state, and pushes the menu onto the UI stack.
- **Inputs:** None (reads `client_status` cvar)
- **Outputs/Return:** `void`
- **Side effects:** Sets `s_status` and `s_rankstatus_message`; calls `RankStatus_MenuInit`, `trap_CL_UI_RankUserReset`, `UI_PushMenu`; may call `UI_ForceMenuOff` for active/spectator status
- **Calls:** `trap_Cvar_VariableValue`, `RankStatus_MenuInit`, `trap_CL_UI_RankUserReset`, `UI_PushMenu`, `UI_ForceMenuOff`
- **Notes:** `QGR_STATUS_NEW` and `QGR_STATUS_PENDING` return silently with `GRANK_FIXME` comments indicating incomplete handling. `QGR_STATUS_NO_USER` comment also flags an inversion ("get this when user exists").

## Control Flow Notes
This file is event-driven UI code with no per-frame update logic. `UI_RankStatusMenu` is called externally (e.g., from `ui_main.c` or a server command handler) after a ranking operation result arrives. It pushes onto the menu stack; `UI_PopMenu` and subsequent pushes in `RankStatus_MenuEvent` handle the exit transition.

## External Dependencies
- **Includes:** `ui_local.h` (pulls in `q_shared.h`, `bg_public.h`, menu framework types, all `trap_*` syscall declarations)
- **Defined elsewhere:**
  - `grank_status_t` and its `QGR_STATUS_*` constants — ranking system types (defined in ranking headers pulled through `ui_local.h`)
  - `trap_Cvar_VariableValue`, `trap_R_RegisterShaderNoMip`, `trap_CL_UI_RankUserReset` — VM syscall stubs (`ui_syscalls.c`)
  - `UI_PopMenu`, `UI_PushMenu`, `UI_ForceMenuOff` — menu stack management (`ui_atoms.c`)
  - `UI_RankingsMenu`, `UI_LoginMenu`, `UI_SignupMenu` — sibling ranking UI screens
  - `Menu_AddItem` — menu framework (`ui_qmenu.c`)
  - `colorRed` — shared color constant (`ui_qmenu.c` / `q_shared.c`)
