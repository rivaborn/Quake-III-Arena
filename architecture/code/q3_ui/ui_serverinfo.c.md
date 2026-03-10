# code/q3_ui/ui_serverinfo.c

## File Purpose
Implements the "Server Info" UI menu in Quake III Arena's legacy q3_ui module. It displays key-value pairs from the current server's config string and provides "Add to Favorites" and "Back" actions.

## Core Responsibilities
- Fetch and display the server's `CS_SERVERINFO` config string as a key-value table
- Vertically center the info table based on the number of lines
- Allow the player to add the current server to the favorites list (cvars `server1`–`serverN`)
- Prevent the "Add to Favorites" action when a local server is running (`sv_running`)
- Pre-cache UI art assets via `trap_R_RegisterShaderNoMip`
- Provide keyboard and mouse event routing through the standard menu framework

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `serverinfo_t` | struct | Aggregates all menu widgets and runtime state (info string, line count) for this screen |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_serverinfo` | `serverinfo_t` (static) | file-static | Singleton state for the server info menu; zeroed on each `UI_ServerInfoMenu` call |
| `serverinfo_artlist` | `char*[]` (static) | file-static | NULL-terminated list of shader paths to pre-cache |

## Key Functions / Methods

### Favorites_Add
- **Signature:** `void Favorites_Add( void )`
- **Purpose:** Adds the current server address (`cl_currentServerAddress`) to the first available `serverN` cvar slot (up to `MAX_FAVORITESERVERS = 16`).
- **Inputs:** None (reads `cl_currentServerAddress` and `server1`–`server16` via cvars)
- **Outputs/Return:** `void`
- **Side effects:** Writes to a `serverN` cvar via `trap_Cvar_Set`; early-returns if address is empty or already present
- **Calls:** `trap_Cvar_VariableStringBuffer`, `Q_stricmp`, `va`, `trap_Cvar_Set`
- **Notes:** Slot selection logic: takes the first slot whose first character is non-numeric (`adrstr[0] < '0' || > '9'`), which is a heuristic for "empty or non-address" entries; the `best` sentinel starts at 0 meaning "not found yet."

### ServerInfo_Event
- **Signature:** `static void ServerInfo_Event( void* ptr, int event )`
- **Purpose:** Callback for menu item activation — `ID_ADD` triggers `Favorites_Add` then pops the menu; `ID_BACK` just pops the menu.
- **Inputs:** `ptr` — pointer to the activated `menucommon_s`; `event` — event type (only `QM_ACTIVATED` is acted upon)
- **Outputs/Return:** `void`
- **Side effects:** May modify favorite server cvars; always calls `UI_PopMenu` on activation
- **Calls:** `Favorites_Add`, `UI_PopMenu`

### ServerInfo_MenuDraw
- **Signature:** `static void ServerInfo_MenuDraw( void )`
- **Purpose:** Custom draw callback; iterates the info string key-value pairs and renders them centered vertically, then delegates to `Menu_Draw` for the standard widget layer.
- **Inputs:** None (reads `s_serverinfo.info`, `s_serverinfo.numlines`)
- **Outputs/Return:** `void`
- **Side effects:** Issues rendering calls via `UI_DrawString`
- **Calls:** `Info_NextPair`, `Q_strcat`, `UI_DrawString`, `Menu_Draw`
- **Notes:** Keys are right-aligned at `SCREEN_WIDTH * 0.50 - 8`; values are left-aligned at `+8`. Line count is capped at 16 in `UI_ServerInfoMenu`, preventing overflow.

### ServerInfo_MenuKey
- **Signature:** `static sfxHandle_t ServerInfo_MenuKey( int key )`
- **Purpose:** Key handler; delegates entirely to the default menu key processor.
- **Inputs:** `key` — raw key code
- **Outputs/Return:** `sfxHandle_t` — sound handle for key feedback
- **Calls:** `Menu_DefaultKey`

### ServerInfo_Cache
- **Signature:** `void ServerInfo_Cache( void )`
- **Purpose:** Pre-registers all art assets in `serverinfo_artlist` with the renderer.
- **Side effects:** Registers shaders via `trap_R_RegisterShaderNoMip`

### UI_ServerInfoMenu
- **Signature:** `void UI_ServerInfoMenu( void )`
- **Purpose:** Entry point; zeroes state, caches art, initializes all menu items, reads and counts the server info string, then pushes the menu.
- **Side effects:** Zeros `s_serverinfo`; calls `trap_GetConfigString(CS_SERVERINFO, ...)`; caps `numlines` at 16; calls `UI_PushMenu`
- **Calls:** `memset`, `ServerInfo_Cache`, `trap_Cvar_VariableValue`, `trap_GetConfigString`, `Info_NextPair`, `Menu_AddItem`, `UI_PushMenu`

## Control Flow Notes
`UI_ServerInfoMenu` is the external entry point called to push this screen onto the menu stack. Each frame, the engine calls the menu's `draw` function pointer (`ServerInfo_MenuDraw`) and routes key events through `ServerInfo_MenuKey`. The screen has no per-frame update logic; all state is populated once at open time from `CS_SERVERINFO`.

## External Dependencies
- `ui_local.h` — menu framework types, trap functions, draw utilities, `MAX_FAVORITESERVERS`
- `trap_GetConfigString` / `CS_SERVERINFO` — defined in engine/qcommon layer
- `Info_NextPair` — defined in `q_shared.c`
- `UI_PushMenu`, `UI_PopMenu`, `Menu_Draw`, `Menu_DefaultKey`, `Menu_AddItem` — defined in `ui_qmenu.c`
- `UI_DrawString` — defined in `ui_atoms.c`
- `trap_R_RegisterShaderNoMip`, `trap_Cvar_*` — syscall stubs defined in `ui_syscalls.c`
