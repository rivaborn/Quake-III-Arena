# code/q3_ui/ui_sppostgame.c

## File Purpose
Implements the single-player postgame menu for Quake III Arena, displayed after a match ends. It orchestrates a three-phase animated sequence: podium presentation, award medal display, then interactive buttons for replay/next/menu navigation.

## Core Responsibilities
- Parse postgame command arguments (scores, ranks, award stats) into menu state
- Drive a three-phase timed presentation (podium → awards → navigation buttons)
- Display and animate per-award medals with sounds
- Evaluate tier/level progression logic to determine the "Next" level destination
- Trigger tier cinematic videos upon tier completion
- Persist best scores and award data via `UI_SetBestScore` / `UI_LogAwardData`
- Register and play winner/loser music and announcement sounds

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `postgameMenuInfo_t` | struct | All state for the postgame menu: phase, timers, scoreboard data, award arrays, win/tier info, menu items |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `postgameMenuInfo` | `postgameMenuInfo_t` | static (file) | Sole instance of postgame menu state |
| `arenainfo` | `char[MAX_INFO_VALUE]` | static (file) | Cached arena info string for current map |
| `ui_medalNames` | `char *[]` | global | Human-readable medal names (6 entries) |
| `ui_medalPicNames` | `char *[]` | global | Shader paths for medal images |
| `ui_medalSounds` | `char *[]` | global | WAV paths for medal announcement sounds |
| `medalLocations` | `int[6]` | static (file) | Fixed X positions for medal icons on screen |

## Key Functions / Methods

### UI_SPPostgameMenu_f
- **Signature:** `void UI_SPPostgameMenu_f( void )`
- **Purpose:** Entry point called by the game engine via console command; parses all postgame data, initializes menu state, and pushes the menu.
- **Inputs:** Arguments via `UI_Argv()` — client count, player client num, award values, per-client rank/score triples.
- **Outputs/Return:** void
- **Side effects:** Writes `postgameMenuInfo`, calls `UI_SetBestScore`, `UI_LogAwardData`, `trap_Key_SetCatcher`, resets `uis.menusp`, plays music, registers sounds.
- **Calls:** `UI_GetArenaInfoByMap`, `UI_SetBestScore`, `UI_TierCompleted`, `UI_LogAwardData`, `UI_GetAwardLevel`, `UI_GetNumSPTiers`, `UI_GetSpecialArenaInfo`, `UI_SPPostgameMenu_Init`, `UI_PushMenu`, `Menu_SetCursorToItem`, `Prepname`, `trap_S_RegisterSound`, `trap_Cmd_ExecuteText`
- **Notes:** Caps client count at `MAX_SCOREBOARD_CLIENTS` (8). `won == -1` means player did not place first. FRAGS award uses a cumulative milestone (per-100) rather than a per-game threshold.

### UI_SPPostgameMenu_MenuDraw
- **Signature:** `static void UI_SPPostgameMenu_MenuDraw( void )`
- **Purpose:** Per-frame draw callback; drives phase transitions by comparing `uis.realtime` against stored start times.
- **Inputs:** None (reads `postgameMenuInfo` and `uis` globals)
- **Outputs/Return:** void
- **Side effects:** Mutates `postgameMenuInfo.phase` and `starttime`; may call `UI_PopMenu` on server ID mismatch; may trigger cinematic via `trap_Cmd_ExecuteText`; activates navigation button items in phase 3.
- **Calls:** `trap_GetConfigString`, `UI_PopMenu`, `UI_DrawProportionalString`, `UI_SPPostgameMenu_DrawAwardsPresentation`, `UI_ShowTierVideo`, `trap_Cvar_Set`, `trap_Cvar_SetValue`, `Menu_Draw`, `UI_SPPostgameMenu_MenuDrawScoreLine`
- **Notes:** Phase 1 lasts ≥5 s (podium); phase 2 runs `numAwards × 2000 ms` then ≥5 s hold; phase 3 activates buttons and optionally plays tier cinematics. Scoreboard scrolls if `ui_spScoreboard` cvar is set.

### UI_SPPostgameMenu_MenuKey
- **Signature:** `static sfxHandle_t UI_SPPostgameMenu_MenuKey( int key )`
- **Purpose:** Intercepts key presses to fast-forward through phases before enabling normal menu navigation.
- **Inputs:** `key` — key code
- **Outputs/Return:** `sfxHandle_t` (0 = silent, or default key sound)
- **Side effects:** Advances `postgameMenuInfo.phase`, sets `ignoreKeysTime` to debounce rapid presses; issues `abort_podium` command in phase 1.
- **Notes:** Blocks `K_ESCAPE`/`K_MOUSE2` in phases 1–2 to prevent accidental dismissal.

### UI_SPPostgameMenu_NextEvent
- **Signature:** `static void UI_SPPostgameMenu_NextEvent( void* ptr, int event )`
- **Purpose:** Handles "Next" button; computes the correct next level respecting tier boundaries and current SP progress.
- **Calls:** `UI_GetCurrentGame`, `UI_GetNumSPTiers`, `UI_GetArenaInfoByNumber`, `UI_SPArena_Start`, `UI_PopMenu`
- **Notes:** If the player hasn't actually won (`won == 0`, training map), next level is forced to 0. Will not advance beyond the player's currently unlocked tier set.

### UI_SPPostgameMenu_Cache
- **Signature:** `void UI_SPPostgameMenu_Cache( void )`
- **Purpose:** Pre-registers all shaders and sounds used by the postgame screen.
- **Side effects:** Calls `trap_R_RegisterShaderNoMip` and `trap_S_RegisterSound` for all medal assets and button art.
- **Notes:** When `com_buildscript` is set, also registers loss/win music and the "youwin" sound to force inclusion in build bundles.

### Prepname
- **Signature:** `static void Prepname( int index )`
- **Purpose:** Fetches, cleans, and truncates a player's name (for podium display) into `postgameMenuInfo.placeNames[index]`.
- **Notes:** Truncates until `UI_ProportionalStringWidth` fits within 256 pixels.

### Notes (trivial helpers)
- `UI_SPPostgameMenu_AgainEvent`: pops menu and issues `map_restart 0`.
- `UI_SPPostgameMenu_MenuEvent`: pops menu and disconnects to level select.
- `UI_SPPostgameMenu_DrawAwardsMedals`: draws up to `max` earned medal icons at fixed X positions.
- `UI_SPPostgameMenu_DrawAwardsPresentation`: draws one award at a time with fade-out alpha and plays its sound once.
- `UI_SPPostgameMenu_MenuDrawScoreLine`: renders a single scoreboard row with rank, name, score.

## Control Flow Notes
`UI_SPPostgameMenu_f` is triggered by a game-side console command at match end. After init, `UI_SPPostgameMenu_MenuDraw` is called every UI frame via the `menu.draw` function pointer. The file owns phase progression entirely within that draw callback; no separate update tick exists.

## External Dependencies
- `ui_local.h` → pulls in `q_shared.h`, `bg_public.h`, `ui_public.h`, `keycodes.h`, all menu/trap declarations
- **Defined elsewhere:** `UI_GetArenaInfoByMap`, `UI_GetArenaInfoByNumber`, `UI_TierCompleted`, `UI_ShowTierVideo`, `UI_GetNumSPTiers`, `UI_GetSpecialArenaInfo`, `UI_SetBestScore`, `UI_LogAwardData`, `UI_GetAwardLevel`, `UI_GetCurrentGame`, `UI_SPArena_Start`, `Menu_*` functions, all `trap_*` syscalls, `uis` global, draw utilities (`UI_DrawProportionalString`, `UI_DrawNamedPic`, `UI_DrawString`)
