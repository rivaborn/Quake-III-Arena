# code/cgame/cg_draw.c

## File Purpose
Implements all 2D and some 3D HUD rendering for the cgame module during active gameplay. It draws the status bar, crosshair, lagometer, scoreboards, team overlays, center prints, and all other screen-space UI elements composited over the 3D world view.

## Core Responsibilities
- Render the player status bar (health, armor, ammo, weapon model icons)
- Draw the crosshair and crosshair entity name labels
- Display team overlay, scores, powerup timers, and pickup notifications
- Render the lagometer (frame interpolation + snapshot latency graph)
- Handle center-print messages with fade timing
- Orchestrate the full 2D draw pass (`CG_Draw2D`) called each frame after 3D scene render
- Drive the top-level `CG_DrawActive` entry point for stereo-aware full-screen rendering

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `lagometer_t` | struct | Ring-buffer storing frame interpolation offsets and snapshot ping/drop samples for the lagometer graph |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `lagometer` | `lagometer_t` | global | Accumulates per-frame and per-snapshot network timing data for lagometer rendering |
| `sortedTeamPlayers` | `int[TEAM_MAXOVERLAY]` | global | Sorted client indices for team overlay display |
| `numSortedTeamPlayers` | `int` | global | Count of valid entries in `sortedTeamPlayers` |
| `systemChat` | `char[256]` | global | System chat message buffer |
| `teamChat1` | `char[256]` | global | Team chat line 1 buffer |
| `teamChat2` | `char[256]` | global | Team chat line 2 buffer |
| `menuScoreboard` | `menuDef_t *` | global (MISSIONPACK only) | Pointer to the parsed scoreboard menu definition |
| `drawTeamOverlayModificationCount` | `int` | global (non-MISSIONPACK) | Tracks when team overlay needs rebuild |

## Key Functions / Methods

### CG_DrawActive
- **Signature:** `void CG_DrawActive(stereoFrame_t stereoView)`
- **Purpose:** Top-level per-frame render entry point; renders the 3D scene then the 2D HUD
- **Inputs:** `stereoView` — center, left, or right eye for stereo separation
- **Outputs/Return:** void
- **Side effects:** Calls `trap_R_RenderScene`, modifies `cg.refdef.vieworg` temporarily for stereo offset
- **Calls:** `CG_DrawInformation`, `CG_DrawTourneyScoreboard`, `CG_TileClear`, `trap_R_RenderScene`, `CG_Draw2D`
- **Notes:** Falls through to info screen if no valid snapshot; handles tournament spectator scoreboard shortcut

### CG_Draw2D
- **Signature:** `static void CG_Draw2D(void)`
- **Purpose:** Orchestrates all 2D HUD drawing for a normal gameplay frame
- **Inputs:** None (reads global `cg`, `cgs`)
- **Outputs/Return:** void
- **Side effects:** Drives all HUD sub-draws; sets `cg.scoreBoardShowing`
- **Calls:** `CG_DrawIntermission`, `CG_DrawSpectator`, `CG_DrawStatusBar`, `CG_DrawCrosshair`, `CG_DrawCrosshairNames`, `CG_DrawWeaponSelect`, `CG_DrawReward`, `CG_DrawVote`, `CG_DrawTeamVote`, `CG_DrawLagometer`, `CG_DrawUpperRight`, `CG_DrawLowerRight`, `CG_DrawLowerLeft`, `CG_DrawFollow`, `CG_DrawWarmup`, `CG_DrawScoreboard`, `CG_DrawCenterString`
- **Notes:** Skips all drawing if `cg.levelShot` or `cg_draw2D == 0`; early returns to intermission path on `PM_INTERMISSION`

### CG_AddLagometerFrameInfo
- **Signature:** `void CG_AddLagometerFrameInfo(void)`
- **Purpose:** Records current frame interpolation offset into the lagometer ring buffer
- **Inputs:** None
- **Side effects:** Writes to `lagometer.frameSamples`, increments `lagometer.frameCount`
- **Calls:** None (reads `cg.time`, `cg.latestSnapshotTime`)

### CG_AddLagometerSnapshotInfo
- **Signature:** `void CG_AddLagometerSnapshotInfo(snapshot_t *snap)`
- **Purpose:** Records snapshot ping or dropped-packet marker into the lagometer ring buffer
- **Inputs:** `snap` — NULL for a dropped packet, otherwise valid snapshot
- **Side effects:** Writes to `lagometer.snapshotSamples`/`snapshotFlags`, increments `lagometer.snapshotCount`

### CG_DrawLagometer
- **Signature:** `static void CG_DrawLagometer(void)`
- **Purpose:** Renders the network quality graph (frame jitter + ping/drop) in lower-right corner
- **Inputs:** None
- **Side effects:** `trap_R_SetColor`, `trap_R_DrawStretchPic` calls; delegates to `CG_DrawDisconnect`
- **Notes:** Skipped on local server (`cgs.localServer`); draws "snc" label when prediction is disabled

### CG_CenterPrint
- **Signature:** `void CG_CenterPrint(const char *str, int y, int charWidth)`
- **Purpose:** Sets up a center-screen message that fades over `cg_centertime` seconds
- **Inputs:** `str` — message text; `y` — vertical center; `charWidth` — character size
- **Side effects:** Writes `cg.centerPrint`, `cg.centerPrintTime`, `cg.centerPrintY`, `cg.centerPrintLines`

### CG_Draw3DModel
- **Signature:** `void CG_Draw3DModel(float x, float y, float w, float h, qhandle_t model, qhandle_t skin, vec3_t origin, vec3_t angles)`
- **Purpose:** Renders a 3D model into a 2D screen rectangle (used for weapon/armor/head icons)
- **Side effects:** `trap_R_ClearScene`, `trap_R_AddRefEntityToScene`, `trap_R_RenderScene`
- **Notes:** No-ops if `cg_draw3dIcons` or `cg_drawIcons` is disabled

### CG_DrawCrosshair
- **Signature:** `static void CG_DrawCrosshair(void)`
- **Purpose:** Draws the crosshair sprite centered on the viewport, with optional health coloring and pickup pulse
- **Side effects:** `trap_R_SetColor`, `trap_R_DrawStretchPic`
- **Notes:** Suppressed in spectator mode and third-person view

### CG_ScanForCrosshairEntity / CG_DrawCrosshairNames
- Combined purpose: ray-cast from view origin to find a client entity under the crosshair; display their name with fade

### CG_DrawTeamOverlay
- **Signature:** `static float CG_DrawTeamOverlay(float y, qboolean right, qboolean upper)`
- **Purpose:** Renders per-teammate health/armor/location/weapon/powerup status panel
- **Returns:** Updated `y` coordinate after the overlay block

## Control Flow Notes
- **Per-frame entry:** `CG_DrawActiveFrame` (in `cg_view.c`) → `CG_DrawActive` → `CG_Draw2D`
- **Lagometer feed:** called from snapshot processing (`CG_AddLagometerSnapshotInfo`) and frame processing (`CG_AddLagometerFrameInfo`)
- **Center print:** set by server command parsing; drawn each frame until timed out
- The file has no init or shutdown logic; all state is in `cg`/`cgs` globals reset on level load

## External Dependencies
- `cg_local.h` — all cgame types, `cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items`, trap declarations
- `ui/ui_shared.h` (MISSIONPACK) — `displayContextDef_t`, `menuDef_t`, `Menu_Paint`, `Menus_FindByName`
- **Defined elsewhere:** `CG_DrawOldScoreboard`, `CG_DrawOldTourneyScoreboard`, `CG_DrawWeaponSelect`, `CG_DrawStringExt`, `CG_DrawBigString`, `CG_FadeColor`, `CG_ColorForHealth`, `CG_AdjustFrom640`, `BG_FindItemForPowerup`, `trap_R_*`, `trap_S_*`, `trap_CM_*`, `g_color_table`, `colorWhite`, `colorBlack`
