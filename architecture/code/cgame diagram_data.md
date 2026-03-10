# code/cgame/cg_consolecmds.c
## File Purpose
Registers and dispatches client-side console commands typed at the local console or bound to keys. It bridges player input (keyboard bindings, console text) to cgame actions such as score display, weapon cycling, team orders, and voice chat.

## Core Responsibilities
- Defines a static dispatch table (`commands[]`) mapping command name strings to handler functions
- Implements `CG_ConsoleCommand` to look up and invoke handlers when the engine forwards an unrecognized command to cgame
- Implements `CG_InitConsoleCommands` to register all commands with the engine for tab-completion
- Handles scoreboard show/hide state and optional score refresh requests
- Provides tell/voice-tell shortcuts targeting crosshair player or last attacker
- Under `MISSIONPACK`: handles HUD reloading, team orders, scoreboard scrolling, and SP win/lose sequences

## External Dependencies
- `cg_local.h` — full cgame state (`cg_t cg`, `cgs_t cgs`), trap declarations, and function prototypes
- `../ui/ui_shared.h` — `menuDef_t`, `Menu_ScrollFeeder`, `String_Init`, `Menu_Reset`
- **Defined elsewhere:** `CG_CrosshairPlayer`, `CG_LastAttacker`, `CG_LoadMenus`, `CG_AddBufferedSound`, `CG_CenterPrint`, `CG_BuildSpectatorString`, `CG_SelectNextPlayer`/`CG_SelectPrevPlayer`, `CG_OtherTeamHasFlag`, `CG_YourTeamHasFlag`, `CG_LoadDeferredPlayers`, all `CG_TestModel_*`/`CG_Zoom*`/`CG_*Weapon_f` functions, all `trap_*` syscalls

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

## External Dependencies
- `cg_local.h` — all cgame types, `cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items`, trap declarations
- `ui/ui_shared.h` (MISSIONPACK) — `displayContextDef_t`, `menuDef_t`, `Menu_Paint`, `Menus_FindByName`
- **Defined elsewhere:** `CG_DrawOldScoreboard`, `CG_DrawOldTourneyScoreboard`, `CG_DrawWeaponSelect`, `CG_DrawStringExt`, `CG_DrawBigString`, `CG_FadeColor`, `CG_ColorForHealth`, `CG_AdjustFrom640`, `BG_FindItemForPowerup`, `trap_R_*`, `trap_S_*`, `trap_CM_*`, `g_color_table`, `colorWhite`, `colorBlack`

# code/cgame/cg_drawtools.c
## File Purpose
Provides low-level 2D rendering helper functions for the cgame module, including coordinate scaling, filled/outlined rectangles, image blitting, character/string rendering, and HUD utility queries. All functions operate in a virtual 640×480 coordinate space and scale to the actual display resolution.

## Core Responsibilities
- Scale 640×480 virtual coordinates to real screen pixels via `cgs.screenXScale`/`screenYScale`
- Draw filled rectangles, bordered rectangles, and textured quads
- Render individual characters and multi-style strings (color codes, shadows, proportional fonts, banner fonts)
- Tile background graphics around a reduced viewport
- Compute time-based fade alpha and team color vectors
- Map health/armor values to a color gradient for HUD display

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere**: `cg` (`cg_t`), `cgs` (`cgs_t`), `g_color_table`, `Q_IsColorString`, `ColorIndex`, `VectorClear`
- **Trap calls**: `trap_R_SetColor`, `trap_R_DrawStretchPic` (renderer syscalls)
- **Constants used**: `FADE_TIME`, `PULSE_DIVISOR`, `PROP_*`, `PROPB_*`, `UI_CENTER/RIGHT/DROPSHADOW/INVERSE/PULSE/SMALLFONT/FORMATMASK`, `ARMOR_PROTECTION`, `BIGCHAR_*`, `SMALLCHAR_*`

# code/cgame/cg_effects.c
## File Purpose
Generates client-side visual effects as local entities, primarily in response to game events such as weapon impacts, player deaths, teleportation, and special item activations. All effects are purely cosmetic and client-local, not networked.

## Core Responsibilities
- Spawn bubble trail local entities for underwater projectiles
- Create smoke puff / blood trail local entities with configurable color, fade, and velocity
- Generate explosion local entities (sprite and model variants)
- Spawn player gib fragments with randomized gravity trajectories
- Handle teleport, score plum, and MissionPack-exclusive effects (Kamikaze, Obelisk, Invulnerability)
- Emit positional sounds for pain and impact events (Obelisk, Invulnerability)

## External Dependencies
- `cg_local.h`: All cgame types (`localEntity_t`, `cg_t`, `cgs_t`, `leType_t`, etc.)
- `cg.time`: Current client render time (global `cg_t`)
- `cgs.media.*`: Preloaded shader/model handles (global `cgs_t`)
- `cgs.glconfig.hardwareType`: GPU capability check for RagePro fallback
- **Defined elsewhere**: `CG_AllocLocalEntity` (`cg_localents.c`), `CG_MakeExplosion` (this file, called by `CG_ObeliskExplode`), `trap_S_StartSound` (syscall layer), `AxisClear`, `RotateAroundDirection`, `VectorNormalize`, `AnglesToAxis` (math library), `axisDefault` (global defined in renderer/shared code)

# code/cgame/cg_ents.c
## File Purpose
Presents server-transmitted snapshot entities to the renderer and sound system every frame. It resolves interpolated/extrapolated positions for all `centity_t` objects and dispatches per-type rendering logic (players, missiles, movers, items, etc.).

## Core Responsibilities
- Compute per-frame lerp/extrapolated origins and angles for all packet entities via `CG_CalcEntityLerpPositions`
- Apply continuous per-entity effects (looping sounds, constant lights) via `CG_EntityEffects`
- Dispatch entity-type-specific rendering through `CG_AddCEntity` (switch on `eType`)
- Attach child render entities to parent model tags (`CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag`)
- Adjust entity positions when riding movers (`CG_AdjustPositionForMover`)
- Drive the auto-rotation state (`cg.autoAngles/autoAxis`) used by all world items
- Submit the local predicted player entity in addition to server-sent entities

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `tr_types.h`, `cg_public.h`
- **Defined elsewhere:** `CG_Player` (`cg_players.c`), `CG_AddRefEntityWithPowerups` (`cg_players.c`), `CG_GrappleTrail` (`cg_weapons.c`), `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta`, `BG_PlayerStateToEntityState` (`bg_misc.c`/`bg_pmove.c`)
- Renderer traps: `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_R_LerpTag`
- Sound traps: `trap_S_UpdateEntityPosition`, `trap_S_AddLoopingSound`, `trap_S_AddRealLoopingSound`, `trap_S_StartSound`
- Math utilities: `VectorCopy/MA/Add/Subtract/Scale/Clear/Normalize2`, `AnglesToAxis`, `MatrixMultiply`, `AxisCopy/Clear`, `RotateAroundDirection`, `PerpendicularVector`, `CrossProduct`, `ByteToDir`, `LerpAngle`

# code/cgame/cg_event.c
## File Purpose
Handles client-side entity event processing at snapshot transitions and playerstate changes. It translates server-generated event codes into audio, visual, and HUD feedback for the local client. This is the primary event dispatch hub for the cgame module.

## Core Responsibilities
- Dispatch `EV_*` events from entity states to appropriate audio/visual handlers
- Display kill obituary messages in the console and center-print frags to the killer
- Handle item pickup notification, weapon selection, and holdable item usage
- Manage movement feedback: footsteps, fall sounds, step smoothing, jump pad effects
- Route CTF/team-mode global sound events with team-context-aware sound selection
- Forward missile impact, bullet, railgun, and shotgun events to weapon effect functions
- Gate event re-firing by tracking `previousEvent` on each `centity_t`

## External Dependencies
- `cg_local.h` — all cgame types, globals (`cg`, `cgs`), and trap declarations.
- `ui/menudef.h` — `VOICECHAT_*` constants (MissionPack only).
- **Defined elsewhere:** `BG_EvaluateTrajectory`, `BG_FindItemForHoldable`, `ByteToDir`, `Info_ValueForKey`, `Q_strncpyz`, `Com_sprintf`, `va`; all `CG_*` effect/weapon functions; all `trap_S_*` sound traps.

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

## External Dependencies
- **Includes**: `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere**: `bg_itemlist` (game shared item table), `CG_ConfigString`, `CG_DrawPic`, `UI_DrawProportionalString`, `trap_R_RegisterShaderNoMip`, `trap_R_RegisterShader`, `trap_R_DrawStretchPic`, `trap_R_SetColor`, `trap_S_RegisterSound`, `trap_UpdateScreen`, `trap_Cvar_VariableStringBuffer`, `Q_strncpyz`, `Q_strrchr`, `Q_CleanStr`, `Info_ValueForKey`, `Com_sprintf`, `va`, `atoi`

# code/cgame/cg_local.h
## File Purpose
Central private header for the Quake III Arena cgame (client-game) module. Defines all major data structures, global state declarations, cvar externs, and function prototypes used across every cgame source file. Acts as the single shared contract binding all cgame subsystems together.

## Core Responsibilities
- Define timing/animation/display constants for client-side visual effects
- Declare `centity_t`, `cg_t`, `cgs_t`, `cgMedia_t`, `clientInfo_t`, `weaponInfo_t`, and related types
- Declare all cgame-module-global extern variables (`cg`, `cgs`, `cg_entities`, `cg_weapons`, etc.)
- Expose all vmCvar externs used by cgame subsystems
- Prototype all public functions across cgame `.c` files (draw, predict, players, weapons, effects, marks, etc.)
- Declare all engine system trap functions (`trap_*`) that bridge the VM to the main executable

## External Dependencies
- `../game/q_shared.h` — shared math, string, entity/player state, cvar, trace types
- `tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t`, etc.)
- `../game/bg_public.h` — gameplay constants, animation enums, `gitem_t`, `pmove_t`, `playerState_t`
- `cg_public.h` — `snapshot_t`, `cgameImport_t`/`cgameExport_t` enums, import API version
- All `trap_*` symbols: defined in engine executable, resolved at VM load time (not in this file)
- `gitem_t bg_itemlist[]`, `Pmove()`: defined in `bg_*.c` shared game code

# code/cgame/cg_localents.c
## File Purpose
Manages a fixed-size pool of client-side "local entities" (smoke puffs, gibs, brass shells, explosions, score plums, etc.) that exist purely on the client and are never synchronized with the server. Every frame, it iterates all active local entities and submits renderer commands appropriate to each entity type.

## Core Responsibilities
- Maintain a pool of 512 `localEntity_t` slots via a doubly-linked active list and a singly-linked free list
- Allocate and free local entities, evicting the oldest active entity when the pool is exhausted
- Simulate fragment physics: trajectory evaluation, collision tracing, bounce/reflect, mark/sound generation, and ground-sinking
- Drive per-type visual update functions (fade, scale, fall, explosion, sprite explosion, score plum, kamikaze, etc.)
- Submit all live local entities to the renderer each frame via `trap_R_AddRefEntityToScene`

## External Dependencies
- **Includes:** `cg_local.h` → `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:**
  - `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — `bg_pmove.c`
  - `CG_SmokePuff`, `CG_ImpactMark` — `cg_effects.c`, `cg_marks.c`
  - `CG_Trace` — `cg_predict.c`
  - `CG_GibPlayer` — `cg_effects.c`
  - `trap_R_AddRefEntityToScene`, `trap_R_AddLightToScene`, `trap_CM_PointContents`, `trap_S_StartSound`, `trap_S_StartLocalSound` — engine syscall layer
  - `cg`, `cgs` — global state in `cg_main.c`

# code/cgame/cg_main.c
## File Purpose
This is the primary entry point and initialization module for the cgame (client-side game) VM module in Quake III Arena. It owns all global cgame state, registers cvars, and orchestrates the full asset precache pipeline during level load.

## Core Responsibilities
- Expose `vmMain()` as the sole entry point from the engine into the cgame VM
- Declare and own all global cgame state (`cg`, `cgs`, `cg_entities`, `cg_weapons`, `cg_items`)
- Register and update all cgame `vmCvar_t` variables via a data-driven table
- Drive the level initialization sequence: sounds → graphics → clients → HUD
- Provide utility functions: `CG_Printf`, `CG_Error`, `CG_Argv`, `CG_ConfigString`
- Implement stub `Com_Error`/`Com_Printf` linkage for shared `q_shared.c`/`bg_*.c` code
- (MISSIONPACK) Load and initialize the script-driven HUD menu system via `displayContextDef_t`

## External Dependencies
- `cg_local.h` — pulls in `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`, and all `trap_*` declarations
- `ui/ui_shared.h` (MISSIONPACK) — `displayContextDef_t`, menu system types, `Init_Display`, `Menu_*`, `PC_*` parse helpers
- **Defined elsewhere:** `CG_DrawActiveFrame`, `CG_ConsoleCommand`, `CG_NewClientInfo`, `CG_RegisterItemVisuals`, `CG_ParseServerinfo`, `CG_SetConfigValues`, `bg_itemlist`, `bg_numItems`, all `trap_*` syscall stubs

# code/cgame/cg_marks.c
## File Purpose
Manages persistent wall mark decals (bullet holes, burn marks, blood splats) and a full particle simulation system for the cgame module. Despite the filename, the file contains two logically separate systems: mark polys and a Ridah-era particle engine that was folded in.

## Core Responsibilities
- Maintain a fixed-size pool of `markPoly_t` nodes using a doubly-linked active list and singly-linked free list
- Project impact decals onto world geometry by clipping a quad against BSP surfaces via `trap_CM_MarkFragments`
- Fade and expire persistent mark polys each frame, submitting survivors to the renderer
- Maintain a fixed-size pool of `cparticle_t` particles (weather, smoke, blood, bubbles, sprites, animations)
- Update and submit particles each frame with physics integration (velocity + acceleration)
- Provide factory functions for spawning typed particles (snow, smoke, sparks, blood, explosions, etc.)

## External Dependencies
- `cg_local.h` — all cgame types, globals (`cg`, `cgs`), and trap declarations
- `trap_CM_MarkFragments` — BSP polygon clipping (defined in engine)
- `trap_R_AddPolyToScene` — renderer polygon submission (defined in engine)
- `trap_R_RegisterShader` — shader registration for particle anim frames (defined in engine)
- `VectorNormalize2`, `PerpendicularVector`, `RotatePointAroundVector`, `CrossProduct`, `VectorMA`, `DotProduct`, `Distance`, `AngleVectors`, `vectoangles` — math utilities (defined in `q_math.c`/`q_shared.c`)
- `cg_addMarks` cvar — gates mark/particle submission; declared in `cg_main.c`
- `cgs.media.energyMarkShader`, `tracerShader`, `smokePuffShader`, `waterBubbleShader` — preloaded media handles

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

# code/cgame/cg_particles.c
## File Purpose
Implements a software particle system for the cgame module, managing a fixed pool of particles that simulate weather (snow, flurry, bubbles), combat effects (blood, smoke, sparks, explosions), and environmental effects (oil slicks, dust). Particles are submitted each frame as raw polygons to the renderer via `trap_R_AddPolyToScene`.

## Core Responsibilities
- Maintain a free-list / active-list pool of `MAX_PARTICLES` (8192) particles
- Initialize and register animated shader sequences used by explosion/anim particles
- Classify particles by type and build camera-aligned or flat polygon geometry each frame
- Apply simple physics (position = origin + vel*t + accel*t²) during the update pass
- Cull expired particles back to the free list; cull distant particles to avoid poly budget overruns
- Provide typed spawn helpers called from other cgame subsystems (weapons, events, entities)

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `cg` (global `cg_t`), `cgs` (global `cgs_t`) — read for time, refdef, snapshot, media handles, GL config
- `trap_R_RegisterShader`, `trap_R_AddPolyToScene` — renderer syscalls (defined in cgame syscall layer)
- `trap_CM_BoxTrace` (via `CG_Trace`) — used in `ValidBloodPool`
- `crandom`, `random`, `VectorMA`, `VectorCopy`, `vectoangles`, `AngleVectors`, `Distance`, `VectorLength`, `VectorNegate`, `VectorClear`, `VectorSet`, `DEG2RAD` — defined in `q_shared`/`q_math`
- `COM_Parse`, `stricmp`, `atoi`, `atof`, `va`, `memset` — standard/engine string utilities
- `CG_ConfigString`, `CG_Printf`, `CG_Error`, `CG_Trace` — defined elsewhere in cgame

# code/cgame/cg_players.c
## File Purpose
Handles all client-side player entity rendering for Quake III Arena, including model/skin/animation loading, deferred client info management, per-frame skeletal animation evaluation, and visual effect attachment (powerups, flags, shadows, sprites).

## Core Responsibilities
- Load and cache per-client 3-part player models (legs/torso/head) with skins and animations
- Parse `animation.cfg` files to populate animation tables for each player model
- Manage deferred loading of client info to avoid hitches during gameplay
- Evaluate and interpolate animation lerp frames for legs, torso, and flag each frame
- Compute per-frame player orientation (yaw swing, pitch, roll lean, pain twitch)
- Assemble and submit the full player refEntity hierarchy to the renderer
- Attach visual effects: powerup overlays, flag models, haste/breath/dust trails, shadow marks, floating sprites

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `COM_Parse`, `Info_ValueForKey`, `Q_stricmp`, `Q_strncpyz`, `VectorCopy/Clear/Set/MA/Normalize`, `DotProduct`, `AnglesToAxis`, `AngleMod`, `BG_EvaluateTrajectory` — defined in shared/game code
- `trap_*` functions — VM syscall stubs defined in `cg_syscalls.c`
- `CG_SmokePuff`, `CG_ImpactMark`, `CG_PositionRotatedEntityOnTag`, `CG_PositionEntityOnTag`, `CG_AddPlayerWeapon` — defined in other cgame modules
- `cgs`, `cg`, `cg_entities` — global state defined in `cg_main.c`

# code/cgame/cg_playerstate.c
## File Purpose
Processes transitions between consecutive `playerState_t` snapshots on the client side, driving audio feedback, visual damage effects, event dispatch, and UI state updates whenever the local player's state changes. Works for both live prediction and demo/follow-cam playback.

## Core Responsibilities
- Compute and set low-ammo warning level (`CG_CheckAmmo`)
- Calculate screen-shake direction and magnitude from incoming damage (`CG_DamageFeedback`)
- Handle respawn bookkeeping (`CG_Respawn`)
- Dispatch playerstate-embedded events into the entity event system (`CG_CheckPlayerstateEvents`)
- Detect and re-fire predicted events that were corrected by the server (`CG_CheckChangedPredictableEvents`)
- Play context-sensitive local/announcer sounds for hits, kills, rewards, timelimit, and fraglimit (`CG_CheckLocalSounds`)
- Orchestrate all of the above on each snapshot transition (`CG_TransitionPlayerState`)

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere:** `CG_EntityEvent`, `CG_PainEvent`, `CG_AddBufferedSound`, `AngleVectors`, `DotProduct`, `VectorSubtract`, `VectorLength`, `trap_S_StartLocalSound`, `cg`, `cgs`, `cg_entities`, `cg_showmiss`

# code/cgame/cg_predict.c
## File Purpose
Generates `cg.predictedPlayerState` each frame by either interpolating between two server snapshots or running local client-side `Pmove` prediction on unacknowledged user commands. Also provides collision query utilities used by the prediction physics.

## Core Responsibilities
- Build a filtered sublist of solid and trigger entities from the current snapshot for efficient collision tests
- Provide `CG_Trace` and `CG_PointContents` wrappers that test against both world BSP and solid entities
- Interpolate player state between two snapshots when prediction is disabled or in demo playback
- Run client-side `Pmove` on all unacknowledged commands to predict the local player's position ahead of server acknowledgement
- Detect and decay prediction errors caused by server-vs-client divergence
- Predict item pickups and trigger interactions (jump pads, teleporters) locally

## External Dependencies
- **Includes:** `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:**
  - `Pmove` — `bg_pmove.c`
  - `BG_EvaluateTrajectory`, `BG_PlayerTouchesItem`, `BG_CanItemBeGrabbed`, `BG_TouchJumpPad`, `BG_AddPredictableEventToPlayerstate`, `PM_UpdateViewAngles` — `bg_*.c`
  - `CG_AdjustPositionForMover`, `CG_TransitionPlayerState` — `cg_ents.c`, `cg_playerstate.c`
  - All `trap_CM_*` functions — cgame syscall layer (`cg_syscalls.c`)
  - `cg`, `cgs`, `cg_entities[]` — `cg_main.c`

# code/cgame/cg_public.h
## File Purpose
Defines the public interface contract between the cgame module (client-side game logic) and the main engine executable. It declares the snapshot data structure and enumerates all syscall IDs for both engine-to-cgame (imported) and cgame-to-engine (exported) function dispatch tables.

## Core Responsibilities
- Define `snapshot_t`, the primary unit of server-state delivery to the client
- Enumerate all engine services available to the cgame VM via `cgameImport_t` trap IDs
- Enumerate all cgame entry points callable by the engine via `cgameExport_t`
- Define `CMD_BACKUP` / `CMD_MASK` constants for the client command ring buffer
- Declare `CGAME_IMPORT_API_VERSION` for ABI compatibility checking
- Declare cgame UI event type constants (`CGAME_EVENT_*`)

## External Dependencies
- `MAX_MAP_AREA_BYTES` — defined in `qcommon/qfiles.h` or `game/q_shared.h`
- `playerState_t` — defined in `game/bg_public.h`
- `entityState_t` — defined in `game/q_shared.h` / `game/bg_public.h`
- `byte`, `qboolean`, `stereoFrame_t` — defined in `game/q_shared.h`
- `SNAPFLAG_*` constants — defined elsewhere (likely `qcommon/qcommon.h`)
- All `cgameImport_t` trap implementations — defined in `client/cl_cgame.c` (`CL_CgameSystemCalls`)
- All `cgameExport_t` entry points — implemented in `cgame/cg_main.c` (`vmMain`)

# code/cgame/cg_scoreboard.c
## File Purpose
Renders the in-game scoreboard overlay for Quake III Arena, including both the standard mid-game scoreboard and the oversized tournament intermission scoreboard. It handles FFA, team, and spectator layouts with fade animations.

## Core Responsibilities
- Draw per-client score rows with bot icons, player heads, flag indicators, and score/ping/time/name text
- Handle adaptive layout switching between normal and interleaved (compact) modes based on player count
- Render ranked team scoreboards in correct lead order (leading team drawn first)
- Display killer name, current rank/score string, and team score comparison at top of screen
- Draw scoreboard column headers (score/ping/time/name icons)
- Render the full-screen tournament scoreboard with giant text for MOTD, server time, and player scores
- Ensure the local client is always visible, appending their row at the bottom if scrolled off

## External Dependencies
- `cg_local.h` — all shared cgame types, globals (`cg`, `cgs`), and function declarations
- **Defined elsewhere:** `CG_DrawFlagModel`, `CG_DrawPic`, `CG_DrawHead`, `CG_FillRect`, `CG_DrawBigString`, `CG_DrawBigStringColor`, `CG_DrawSmallStringColor`, `CG_DrawStringExt`, `CG_DrawStrlen`, `CG_FadeColor`, `CG_PlaceString`, `CG_DrawTeamBackground`, `CG_LoadDeferredPlayers`, `CG_ConfigString`, `trap_SendClientCommand`, `Com_Printf`, `Com_sprintf`
- Constants `SB_NORMAL_HEIGHT`, `SB_INTER_HEIGHT`, `SB_MAXCLIENTS_NORMAL`, `SB_MAXCLIENTS_INTER`, `SB_SCORELINE_X`, etc. are all `#define`d locally in this file.

# code/cgame/cg_servercmds.c
## File Purpose
Handles reliably-sequenced text commands sent by the server to the cgame module. All commands are processed at snapshot transition time, guaranteeing a valid snapshot is present. Also manages the voice chat system including parsing, buffering, and playback.

## Core Responsibilities
- Dispatch incoming server commands (`cp`, `cs`, `print`, `chat`, `tchat`, `scores`, `tinfo`, `map_restart`, etc.) via `CG_ServerCommand`
- Parse and apply score data (`CG_ParseScores`) and team overlay info (`CG_ParseTeamInfo`)
- Parse and cache server configuration strings (`CG_ParseServerinfo`, `CG_SetConfigValues`)
- Handle config-string change notifications and re-register models/sounds/client info accordingly
- Load, parse, and look up character voice chat files (`.voice`, `.vc`)
- Buffer and throttle voice chat playback with a ring buffer
- Handle map restarts, warmup transitions, and shader remapping

## External Dependencies
- `cg_local.h` — all cgame types (`cg_t`, `cgs_t`, `clientInfo_t`, trap functions, cvars)
- `ui/menudef.h` — `VOICECHAT_*` string constants and UI owner-draw defines
- **Defined elsewhere:** `CG_ConfigString`, `CG_Argv`, `CG_StartMusic`, `CG_NewClientInfo`, `CG_BuildSpectatorString`, `CG_InitLocalEntities`, `CG_InitMarkPolys`, `CG_ClearParticles`, `CG_SetScoreSelection`, `CG_ShowResponseHead`, `CG_LoadDeferredPlayers`, `COM_ParseExt`, `Info_ValueForKey`, all `trap_*` syscalls

# code/cgame/cg_snapshot.c
## File Purpose
Manages the client-side snapshot pipeline, advancing the simulation clock by transitioning between server-delivered game state snapshots. It handles initial snapshot setup, interpolation state tracking, entity transitions, and teleport detection — all without necessarily firing every rendered frame.

## Core Responsibilities
- Read new snapshots from the client system into a double-buffered slot
- Initialize all entity state on the very first snapshot (or map restart)
- Transition `cg.nextSnap` → `cg.snap` when simulation time crosses the boundary
- Set `centity_t.interpolate` flags so the renderer knows whether to lerp or snap entities
- Detect teleport events (both entity-level and playerstate-level) and suppress interpolation accordingly
- Fire entity and playerstate events during snapshot transitions
- Record lagometer data for dropped/received snapshots

## External Dependencies

- **Includes:** `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere:**
  - `trap_GetCurrentSnapshotNumber`, `trap_GetSnapshot` — client system traps
  - `BG_PlayerStateToEntityState` — `bg_misc.c`
  - `CG_BuildSolidList` — `cg_predict.c`
  - `CG_ExecuteNewServerCommands` — `cg_servercmds.c`
  - `CG_Respawn`, `CG_TransitionPlayerState` — `cg_playerstate.c`
  - `CG_CheckEvents` — `cg_events.c`
  - `CG_ResetPlayerEntity` — `cg_players.c`
  - `CG_AddLagometerSnapshotInfo` — `cg_draw.c`
  - `cg`, `cgs`, `cg_entities` — `cg_main.c`

# code/cgame/cg_syscalls.c
## File Purpose
Implements the cgame module's system call interface for the DLL build path. Each `trap_*` function wraps a variadic `syscall` function pointer that dispatches into the engine using integer opcode identifiers defined in `cg_public.h`.

## Core Responsibilities
- Receive and store the engine-provided syscall dispatcher via `dllEntry`
- Expose typed `trap_*` wrappers for every engine service the cgame module needs
- Convert `float` arguments to `int`-width bit-reinterpretations via `PASSFLOAT` before passing through the integer-only syscall ABI
- Cover all engine subsystems: console, cvar, filesystem, collision, sound, renderer, input, cinematic, and snapshot/game-state retrieval

## External Dependencies
- **Includes:** `cg_local.h` → transitively pulls in `q_shared.h`, `tr_types.h`, `bg_public.h`, `cg_public.h`
- **Defined elsewhere:** All `CG_*` opcode constants (e.g., `CG_PRINT`, `CG_R_RENDERSCENE`) — defined in `cg_public.h`; all struct types (`trace_t`, `refEntity_t`, `snapshot_t`, `glconfig_t`, etc.) — defined in shared/renderer headers; `QDECL` calling-convention macro — from `q_shared.h`

# code/cgame/cg_view.c
## File Purpose
Sets up all 3D rendering parameters (view origin, view angles, FOV, viewport rect) each frame and issues the final render call. It is the central per-frame orchestration point for the cgame's visual output.

## Core Responsibilities
- Compute viewport rectangle based on `cg_viewsize` cvar
- Offset first-person or third-person view with bobbing, damage kick, step smoothing, duck smoothing, and land bounce
- Calculate FOV with zoom interpolation and underwater warp
- Build and submit the `refdef_t` to the renderer via `CG_DrawActive`
- Add all scene entities (packet entities, marks, particles, local entities, view weapon, test model)
- Manage a circular sound buffer for announcer/sequential sounds
- Emit powerup-expiry warning sounds
- Provide developer model-testing commands (`testmodel`, `testgun`, frame/skin cycling)

## External Dependencies
- `cg_local.h` — all cgame types, `cg`, `cgs`, cvars, trap declarations
- `trap_R_*` — renderer scene API (defined in engine)
- `trap_S_*` — sound API (defined in engine)
- `CG_DrawActive` — defined in `cg_draw.c`
- `CG_PredictPlayerState`, `CG_Trace`, `CG_PointContents` — defined in `cg_predict.c`
- `CG_AddPacketEntities` — defined in `cg_ents.c`
- `CG_AddViewWeapon` — defined in `cg_weapons.c`
- `CG_PlayBufferedVoiceChats` — defined in `cg_servercmds.c`
- `AnglesToAxis`, `VectorMA`, `DotProduct`, `AngleVectors` — math utilities from `q_math.c`

# code/cgame/cg_weapons.c
## File Purpose
Client-side weapon visualization module for Quake III Arena. Handles all weapon-related rendering, effects, and input, including view weapon display, projectile trails, muzzle flashes, impact effects, shell ejection, and weapon selection UI.

## Core Responsibilities
- Register and cache weapon/item media (models, shaders, sounds) at level load
- Render the first-person view weapon with bobbing, FOV offset, and animation mapping
- Render world-space weapon models attached to player entities (with powerup overlays)
- Emit per-weapon trail effects (rocket smoke, rail rings, plasma sparks, grapple beam)
- Spawn muzzle flash, dynamic light, and brass ejection local entities on fire events
- Resolve hitscan impact effects (explosions, marks, sounds) for all weapon types
- Simulate shotgun pellet spread client-side (matching server seed) for decals/sounds
- Manage weapon cycling (next/prev/direct select) and on-screen weapon selection HUD

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `BG_EvaluateTrajectory` — defined in `bg_misc.c`
- `CG_AllocLocalEntity`, `CG_SmokePuff`, `CG_MakeExplosion`, `CG_BubbleTrail`, `CG_Bleed`, `CG_ImpactMark`, `CG_ParticleExplosion` — defined in other cgame modules
- `CG_PositionEntityOnTag`, `CG_PositionRotatedEntityOnTag` — `cg_ents.c`
- `trap_R_*`, `trap_S_*`, `trap_CM_*` — VM syscall stubs (`cg_syscalls.c`)
- `axisDefault`, `vec3_origin` — defined in `q_math.c` / `q_shared.c`

# code/cgame/tr_types.h
## File Purpose
Defines the shared renderer interface types used by both the client-game (cgame) module and the renderer. It establishes the data structures and constants that describe renderable entities, scene definitions, and OpenGL hardware configuration.

## Core Responsibilities
- Define render entity types and the `refEntity_t` descriptor passed to the renderer
- Define `refdef_t`, the per-frame scene/camera description
- Define OpenGL capability and configuration types (`glconfig_t`)
- Declare bit-flag constants for render effects (`RF_*`) and render definition flags (`RDF_*`)
- Establish hard limits on dynamic lights and renderable entities
- Define polygon vertex and polygon types for decal/effect geometry

## External Dependencies
- **Defined elsewhere:** `vec3_t`, `qhandle_t`, `qboolean`, `byte`, `MAX_STRING_CHARS`, `BIG_INFO_STRING`, `MAX_MAP_AREA_BYTES` — all from `q_shared.h`
- Driver name macros (`_3DFX_DRIVER_NAME`, `OPENGL_DRIVER_NAME`) conditionalized on `Q3_VM` and `_WIN32` platform defines
- `MAX_DLIGHTS 32` is a hard architectural limit because dlight influence is stored as a 32-bit surface bitmask; `MAX_ENTITIES 1023` is constrained by drawsurf sort-key bit packing in the renderer

