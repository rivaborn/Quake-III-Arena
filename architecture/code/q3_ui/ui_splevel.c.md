# code/q3_ui/ui_splevel.c

## File Purpose
Implements the single-player level selection menu for Quake III Arena, allowing players to browse tier-based arena sets, select maps, view completion status, and navigate to the skill selection screen.

## Core Responsibilities
- Initialize and layout the level select menu with up to 4 level thumbnail bitmaps per tier
- Handle tier navigation via left/right arrow buttons
- Display player icon, awards/medals, and bot opponent portraits
- Track and display level completion status with skill-rated completion images
- Handle special-case tiers (training and final) with single-map display
- Provide reset-game confirmation flow and custom/skirmish navigation

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `levelMenuInfo_t` | struct | All state for the level select menu: menu framework, bitmaps, level data, player/bot info, awards |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `levelMenuInfo` | `levelMenuInfo_t` | static (file) | Singleton menu state |
| `selectedArenaSet` | `int` | static (file) | Currently browsed tier index |
| `selectedArena` | `int` | static (file) | Currently selected arena within tier (-1 = none) |
| `currentSet` | `int` | static (file) | Highest unlocked tier |
| `currentGame` | `int` | static (file) | Current game index within unlocked tier |
| `trainingTier` | `int` | static (file) | Index of training special tier (-1 if none) |
| `finalTier` | `int` | static (file) | Index of final special tier |
| `minTier` | `int` | static (file) | Minimum browseable tier |
| `maxTier` | `int` | static (file) | Maximum browseable tier |

## Key Functions / Methods

### PlayerIcon
- **Signature:** `static void PlayerIcon( const char *modelAndSkin, char *iconName, int iconNameMaxSize )`
- **Purpose:** Resolves the icon TGA path for a player model/skin string.
- **Inputs:** Combined `model/skin` string, output buffer, buffer size
- **Outputs/Return:** Writes icon path into `iconName`; falls back to `icon_default.tga` if skin-specific icon not found
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` to test existence
- **Calls:** `Q_strncpyz`, `Q_strrchr`, `Com_sprintf`, `trap_R_RegisterShaderNoMip`, `Q_stricmp`

### UI_SPLevelMenu_SetBots
- **Signature:** `static void UI_SPLevelMenu_SetBots( void )`
- **Purpose:** Parses the `bots` key from the selected arena info and populates `levelMenuInfo.botPics` and `levelMenuInfo.botNames`.
- **Inputs:** None (reads `levelMenuInfo.selectedArenaInfo`)
- **Outputs/Return:** Modifies `levelMenuInfo.numBots`, `.botPics[]`, `.botNames[]`
- **Side effects:** Calls renderer to register bot icon shaders
- **Calls:** `UI_GetBotInfoByName`, `Info_ValueForKey`, `PlayerIconHandle`, `Q_strncpyz`, `Q_CleanStr`
- **Notes:** No-ops if `selectedArenaSet > currentSet` (locked tier)

### UI_SPLevelMenu_SetMenuArena
- **Signature:** `static void UI_SPLevelMenu_SetMenuArena( int n, int level, const char *arenaInfo )`
- **Purpose:** Configures one map slot (index `n`) with name, score, levelshot path, and enabled/grayed state.
- **Side effects:** Modifies `levelMenuInfo.levelPicNames[n]`, `levelScores[n]`, `item_maps[n]` flags/shader

### UI_SPLevelMenu_SetMenuItems
- **Signature:** `static void UI_SPLevelMenu_SetMenuItems( void )`
- **Purpose:** Refreshes all menu items for the currently selected tier, handling training/final special tiers vs. normal 4-arena tiers; enables/disables navigation arrows.
- **Calls:** `UI_SPLevelMenu_SetMenuArena`, `UI_SPLevelMenu_SetBots`, `UI_GetArenaInfoByNumber`, `UI_GetSpecialArenaInfo`, `Bitmap_Init`, `trap_Cvar_SetValue`

### UI_SPLevelMenu_MenuDraw
- **Signature:** `static void UI_SPLevelMenu_MenuDraw( void )`
- **Purpose:** Per-frame custom draw callback; renders player name, levelshots with selection/focus highlights, map name, bot portraits, award levels, and tier label.
- **Side effects:** Calls `UI_PopMenu`/`UI_SPLevelMenu()` if `reinit` flag is set; reads `model` and `name` cvars each frame to detect changes
- **Calls:** `Menu_Draw`, `UI_DrawHandlePic`, `UI_DrawProportionalString`, `trap_R_SetColor`, `trap_Cvar_VariableStringBuffer`

### UI_SPLevelMenu_Init
- **Signature:** `static void UI_SPLevelMenu_Init( void )`
- **Purpose:** Allocates and initializes all menu items, registers assets, reads saved `ui_spSelection`, then calls `UI_SPLevelMenu_SetMenuItems`.
- **Side effects:** `memset` clears `levelMenuInfo`; populates `awardLevels`, registers all bitmaps, calls `UI_SPLevelMenu_Cache`

### UI_SPLevelMenu
- **Signature:** `void UI_SPLevelMenu( void )`
- **Purpose:** Public entry point; computes `currentSet`/`currentGame` from save data, resolves training/final tier boundaries, calls `UI_SPLevelMenu_Init`, pushes menu.
- **Calls:** `UI_GetSpecialArenaInfo`, `UI_GetNumSPTiers`, `UI_GetCurrentGame`, `UI_GetNumSPArenas`, `UI_SPLevelMenu_Init`, `UI_PushMenu`, `Menu_SetCursorToItem`

### UI_SPLevelMenu_ResetAction / UI_SPLevelMenu_ResetEvent
- **Purpose:** Confirms and executes a full single-player progress reset via `UI_NewGame()`, then re-launches the menu.
- **Notes:** Uses the `UI_ConfirmMenu` two-step dialog pattern.

## Control Flow Notes
This file is a **menu module** with no frame/render/update loop of its own. It hooks into the UI system via:
- `UI_SPLevelMenu()` — called externally to push the menu onto the menu stack
- `UI_SPLevelMenu_MenuDraw` — registered as `menu.draw`, called each UI refresh frame
- Event callbacks (`LevelEvent`, `LeftArrowEvent`, etc.) — invoked by the menu framework on `QM_ACTIVATED`
- `UI_SPLevelMenu_ReInit()` — sets a deferred reinit flag consumed on the next draw call

## External Dependencies
- `ui_local.h` → `q_shared.h`, `bg_public.h`, `tr_types.h`, `ui_public.h`
- **Defined elsewhere:** `UI_GetArenaInfoByNumber`, `UI_GetSpecialArenaInfo`, `UI_GetBotInfoByName`, `UI_GetBestScore`, `UI_GetAwardLevel`, `UI_GetCurrentGame`, `UI_NewGame`, `UI_GetNumSPTiers`, `UI_GetNumSPArenas`, `UI_SPSkillMenu`, `UI_StartServerMenu`, `UI_PlayerSettingsMenu`, `UI_ConfirmMenu`, `Menu_Draw`, `Menu_AddItem`, `Bitmap_Init`, `Menu_SetCursorToItem`, `UI_PushMenu`, `UI_PopMenu`, all `trap_*` syscalls, `uis` global, `ui_medalPicNames[]`, `ui_medalSounds[]`, `ARENAS_PER_TIER`, `PULSE_DIVISOR`
