# code/cgame/cg_newdraw.c

## File Purpose
MissionPack (Team Arena)-exclusive HUD drawing module for the cgame client. It implements all "owner draw" HUD element renderers for team game UI elements (health, armor, flags, team overlay, spectator ticker, medals, etc.) and handles mouse/keyboard input routing to the UI display system.

## Core Responsibilities
- Render individual HUD elements via a central `CG_OwnerDraw` dispatch function keyed on owner-draw enum constants
- Display team-specific overlays: selected player health/armor/status/weapon/head, flag status, team scores
- Manage team-ordered player selection (`CG_SelectNextPlayer`, `CG_SelectPrevPlayer`) and pending order dispatch
- Animate the local player's head portrait with damage reaction and idle bobbing
- Draw the scrolling spectator ticker and team chat/system chat areas
- Draw end-of-round medal statistics (accuracy, assists, gauntlet, captures, etc.)
- Route mouse movement and key events to the shared UI `Display_*` system

## Key Types / Data Structures

None defined locally — all types come from `cg_local.h` and `ui_shared.h`.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `drawTeamOverlayModificationCount` | `int` | global | Tracks when the team overlay needs a refresh; set to -1 at startup |
| `healthColors` | `float[4][4]` | static (file), inside `#ifndef MISSIONPACK` guard | Color table for health states — guarded out in MISSIONPACK builds, effectively dead code here |

> `systemChat`, `teamChat1`, `teamChat2` and `sortedTeamPlayers`/`numSortedTeamPlayers` are declared `extern` in `cg_local.h` and defined in `cg_draw.c`.

## Key Functions / Methods

### CG_OwnerDraw
- **Signature:** `void CG_OwnerDraw(float x, float y, float w, float h, float text_x, float text_y, int ownerDraw, int ownerDrawFlags, int align, float special, float scale, vec4_t color, qhandle_t shader, int textStyle)`
- **Purpose:** Central dispatch for all HUD owner-draw elements; called by the UI menu system to render any cgame-owned widget.
- **Inputs:** Screen rect, draw type enum, flags, alignment, color, shader, text style.
- **Outputs/Return:** void; renders to screen.
- **Side effects:** Calls into many `CG_Draw*` helpers which call `trap_R_SetColor`, `CG_DrawPic`, `CG_Text_Paint`.
- **Calls:** All `CG_Draw*` static helpers in this file.
- **Notes:** Exits early if `cg_drawStatus.integer == 0`. The `ownerDrawFlags` visibility check is commented out.

### CG_CheckOrderPending
- **Signature:** `void CG_CheckOrderPending()`
- **Purpose:** If a team order is pending, issues the appropriate voice chat / console command to the selected player or whole team before player selection changes.
- **Inputs:** None (reads `cgs.orderPending`, `cgs.currentOrder`, `cg_currentSelectedPlayer`).
- **Outputs/Return:** void.
- **Side effects:** Calls `trap_SendConsoleCommand`; clears `cgs.orderPending`.
- **Calls:** `trap_SendConsoleCommand`, `va`.
- **Notes:** Only runs for `GT_CTF`+; does nothing for non-team gametypes.

### CG_MouseEvent
- **Signature:** `void CG_MouseEvent(int x, int y)`
- **Purpose:** Handles mouse delta input, clamps cursor to 640×480, updates cursor icon, and forwards to `Display_MouseMove`.
- **Inputs:** Mouse delta x, y.
- **Side effects:** Modifies `cgs.cursorX/Y`, `cgs.activeCursor`, `cgs.capturedItem` via display system.
- **Calls:** `Display_CursorType`, `Display_MouseMove`, `trap_Key_SetCatcher`.

### CG_KeyEvent
- **Signature:** `void CG_KeyEvent(int key, qboolean down)`
- **Purpose:** Routes key events to the UI display system; releases key catcher when player is in normal movement.
- **Inputs:** Key code, down/up state.
- **Side effects:** May capture/release items via `Display_CaptureItem`; calls `CG_EventHandling`.
- **Calls:** `CG_EventHandling`, `trap_Key_SetCatcher`, `Display_HandleKey`, `Display_CaptureItem`.

### CG_DrawNewTeamInfo
- **Signature:** `void CG_DrawNewTeamInfo(rectDef_t *rect, float text_x, float text_y, float scale, vec4_t color, qhandle_t shader)`
- **Purpose:** Renders the team overlay panel showing up to 8 teammates with powerup icons, health indicator, task icon, name, and location.
- **Side effects:** Multiple `CG_DrawPic`, `CG_Text_Paint_Limit` calls; reads `CG_ConfigString` for location names.
- **Calls:** `CG_Text_Width`, `CG_DrawPic`, `CG_GetColorForHealth`, `CG_StatusHandle`, `CG_Text_Paint_Limit`, `BG_FindItemForPowerup`, `trap_R_RegisterShader`, `trap_R_SetColor`.

### CG_DrawTeamSpectators
- **Signature:** `void CG_DrawTeamSpectators(rectDef_t *rect, float scale, vec4_t color, qhandle_t shader)`
- **Purpose:** Scrolls the spectator name list horizontally across the given rect, advancing one character per 10 ms tick.
- **Side effects:** Mutates `cg.spectatorPaintX`, `cg.spectatorPaintX2`, `cg.spectatorOffset`, `cg.spectatorTime`.
- **Calls:** `CG_Text_Width`, `CG_Text_Paint_Limit`.

### CG_DrawMedal
- **Signature:** `void CG_DrawMedal(int ownerDraw, rectDef_t *rect, float scale, vec4_t color, qhandle_t shader)`
- **Purpose:** Renders a post-game medal icon with its numeric value (or "Wow" for perfect), dimming the icon if value is zero.
- **Side effects:** `trap_R_SetColor`, `CG_DrawPic`, `CG_Text_Paint`.

### CG_OwnerDrawVisible
- **Signature:** `qboolean CG_OwnerDrawVisible(int flags)`
- **Purpose:** Returns whether a given owner-draw element should be shown, based on gametype, flag states, player health, team, etc.
- **Notes:** Evaluates flags sequentially (not exclusive OR); some flags like `CG_SHOW_DURINGINCOMINGVOICE` are no-ops (empty body).

### CG_Text_Paint_Limit
- **Signature:** `static void CG_Text_Paint_Limit(float *maxX, float x, float y, float scale, vec4_t color, const char *text, float adjust, int limit)`
- **Purpose:** Paints text glyph-by-glyph, stopping when the running x position would exceed `*maxX`; updates `*maxX` to the last painted x.
- **Notes:** Handles inline color codes (`Q_IsColorString`). Used for width-bounded team overlay columns.

## Control Flow Notes
This file has **no init or frame entry point** of its own. It is called during the HUD render phase initiated from `cg_draw.c`/`CG_DrawActiveFrame`. `CG_OwnerDraw` is registered as a callback in the `displayContextDef_t` structure (`cgDC`) and invoked by the shared UI menu system during `Menu_PaintAll`. `CG_MouseEvent` and `CG_KeyEvent` are called from `cg_main.c` in response to input traps.

## External Dependencies
- **Includes:** `cg_local.h`, `../ui/ui_shared.h`
- **External symbols used but defined elsewhere:**
  - `cgDC` (`displayContextDef_t`) — defined in `cg_main.c`
  - `sortedTeamPlayers[]`, `numSortedTeamPlayers` — defined in `cg_draw.c`
  - `systemChat`, `teamChat1`, `teamChat2` — defined in `cg_draw.c`
  - `cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items` — cgame globals
  - `BG_FindItemForPowerup` — game/bg_misc.c
  - `Display_*`, `Menus_*` — ui/ui_shared.c
  - All `trap_*` functions — cgame VM syscall stubs
