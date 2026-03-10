# code/cgame/cg_info.c

## File Purpose
Implements the loading screen (info screen) displayed while a Quake III Arena level is being loaded. It renders a level screenshot background, player/item icons accumulated during asset loading, and various server/game metadata strings.

## Core Responsibilities
- Accumulate player and item icon handles as clients and items are registered during map load
- Display a loading progress string updated in real time via `trap_UpdateScreen`
- Render the level screenshot backdrop with a detail texture overlay
- Draw server metadata: hostname, pure-server status, MOTD, map message, cheat warning
- Display game type and rule limits (timelimit, fraglimit, capturelimit)
- Register player model icons and, in single-player, pre-cache personality announce sounds

## Key Types / Data Structures
None (no locally defined types; uses `gitem_t`, `qhandle_t`, `cg_t`, `cgs_t` from headers).

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `loadingPlayerIconCount` | `int` | static (file) | Running count of registered player icons |
| `loadingItemIconCount` | `int` | static (file) | Running count of registered item icons |
| `loadingPlayerIcons` | `qhandle_t[16]` | static (file) | Shader handles for up to 16 player icons |
| `loadingItemIcons` | `qhandle_t[26]` | static (file) | Shader handles for up to 26 item icons |

## Key Functions / Methods

### CG_DrawLoadingIcons
- Signature: `static void CG_DrawLoadingIcons(void)`
- Purpose: Draws accumulated player and item icons at fixed screen positions.
- Inputs: None (reads file-static arrays).
- Outputs/Return: None.
- Side effects: Issues `CG_DrawPic` render calls.
- Calls: `CG_DrawPic`
- Notes: Player icons laid out in a single horizontal row (y=284); item icons wrap after 13 per row (y=360, y=400).

### CG_LoadingString
- Signature: `void CG_LoadingString(const char *s)`
- Purpose: Updates the current loading status text and forces an immediate screen refresh.
- Inputs: `s` — status string to display.
- Outputs/Return: None.
- Side effects: Writes to `cg.infoScreenText`; calls `trap_UpdateScreen` (forces a frame flush).
- Calls: `Q_strncpyz`, `trap_UpdateScreen`
- Notes: The `trap_UpdateScreen` call is the only place outside the normal frame loop that triggers a render.

### CG_LoadingItem
- Signature: `void CG_LoadingItem(int itemNum)`
- Purpose: Registers an item's icon shader and updates the loading string with the item's pickup name.
- Inputs: `itemNum` — index into `bg_itemlist`.
- Outputs/Return: None.
- Side effects: Appends to `loadingItemIcons`; calls `CG_LoadingString`.
- Calls: `trap_R_RegisterShaderNoMip`, `CG_LoadingString`

### CG_LoadingClient
- Signature: `void CG_LoadingClient(int clientNum)`
- Purpose: Registers a connecting client's player model icon with three fallback paths, optionally pre-caches a personality announce sound, and updates the loading string.
- Inputs: `clientNum` — index of the connecting client.
- Outputs/Return: None.
- Side effects: Appends to `loadingPlayerIcons`; may call `trap_S_RegisterSound`; calls `CG_LoadingString`.
- Calls: `CG_ConfigString`, `Info_ValueForKey`, `Q_strncpyz`, `Q_strrchr`, `Com_sprintf`, `trap_R_RegisterShaderNoMip`, `Q_CleanStr`, `trap_S_RegisterSound`, `va`, `CG_LoadingString`
- Notes: Falls back first to `models/players/characters/<model>/`, then to `DEFAULT_MODEL/default`. Sound is only registered for `GT_SINGLE_PLAYER`.

### CG_DrawInformation
- Signature: `void CG_DrawInformation(void)`
- Purpose: Renders the complete loading screen: level screenshot, detail overlay, loading icons, and all text information strings.
- Inputs: None (reads global `cg`, `cgs`, config strings).
- Outputs/Return: None.
- Side effects: Issues multiple renderer and UI draw calls; reads cvars via `trap_Cvar_VariableStringBuffer`.
- Calls: `CG_ConfigString`, `Info_ValueForKey`, `trap_R_RegisterShaderNoMip`, `trap_R_SetColor`, `CG_DrawPic`, `trap_R_RegisterShader`, `trap_R_DrawStretchPic`, `CG_DrawLoadingIcons`, `UI_DrawProportionalString`, `trap_Cvar_VariableStringBuffer`, `atoi`, `va`
- Notes: Server info lines (hostname, pure, MOTD) are suppressed when `sv_running` is active (local game). MISSIONPACK game types (`GT_1FCTF`, `GT_OBELISK`, `GT_HARVESTER`) are conditionally compiled.

## Control Flow Notes
This file is active exclusively during the **map load phase**. `CG_LoadingClient` and `CG_LoadingItem` are called by other cgame subsystems as each asset is registered; each call ends with `CG_LoadingString`, which triggers `trap_UpdateScreen` to repaint the screen mid-load. `CG_DrawInformation` is called by the cgame's main draw path whenever `cg.loading` is true, replacing the normal in-game frame. The file-static icon arrays are reset implicitly when the cgame module is unloaded and reloaded on every level change.

## External Dependencies
- **Includes**: `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere**: `bg_itemlist` (game shared item table), `CG_ConfigString`, `CG_DrawPic`, `UI_DrawProportionalString`, `trap_R_RegisterShaderNoMip`, `trap_R_RegisterShader`, `trap_R_DrawStretchPic`, `trap_R_SetColor`, `trap_S_RegisterSound`, `trap_UpdateScreen`, `trap_Cvar_VariableStringBuffer`, `Q_strncpyz`, `Q_strrchr`, `Q_CleanStr`, `Info_ValueForKey`, `Com_sprintf`, `va`, `atoi`
