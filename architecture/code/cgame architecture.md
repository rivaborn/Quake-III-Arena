# Subsystem Overview

## Purpose
The `cgame` subsystem is the client-side game logic VM module for Quake III Arena. It consumes server-delivered snapshots and user input to produce all client-visible output: 3D scene population, 2D HUD rendering, local entity simulation, client-side movement prediction, and audio/visual event feedback. It runs as an isolated QVM (or DLL) and communicates with the engine exclusively through a typed `trap_*` syscall interface.

## Key Files

| File | Role |
|---|---|
| `cg_main.c` | VM entry point (`vmMain`), global state owner, cvar registration, level init/precache orchestration |
| `cg_local.h` | Central private header; all shared types (`cg_t`, `cgs_t`, `centity_t`, `clientInfo_t`, etc.), extern declarations, and `trap_*` prototypes |
| `cg_public.h` | Engine↔cgame ABI contract; `snapshot_t`, `cgameImport_t`/`cgameExport_t` opcode enums, ABI version |
| `cg_syscalls.c` | DLL-path syscall dispatch layer; wraps every `trap_*` call via integer opcode ABI |
| `cg_snapshot.c` | Snapshot pipeline; double-buffered snap advancement, entity/playerstate transitions, teleport suppression |
| `cg_predict.c` | Client-side `Pmove` prediction, snapshot interpolation, solid entity list, `CG_Trace`/`CG_PointContents` |
| `cg_view.c` | Per-frame 3D setup; view origin/angles/FOV, `refdef_t` submission, scene entity assembly |
| `cg_draw.c` | Full 2D HUD draw pass; status bar, crosshair, lagometer, center prints, scoreboards, `CG_DrawActive` |
| `cg_drawtools.c` | Low-level 2D helpers; 640×480→screen coordinate scaling, rectangles, image blits, string renderers |
| `cg_ents.c` | Per-frame packet entity dispatch; lerp/extrapolation, type-specific rendering, tag attachment |
| `cg_players.c` | Player model/skin/animation loading, per-frame skeletal eval, powerup/flag/effect attachment |
| `cg_playerstate.c` | Playerstate snapshot transitions; damage feedback, ammo warnings, event re-fire, local sounds |
| `cg_event.c` | Entity event dispatch hub; translates `EV_*` codes to audio, visual, and HUD feedback |
| `cg_weapons.c` | Weapon media registration, view weapon render, muzzle flash, trails, impact effects, weapon select UI |
| `cg_effects.c` | Cosmetic local entity spawners; explosions, gibs, smoke, bubble trails, teleport effects |
| `cg_localents.c` | Fixed-pool (512) local entity manager; fragment physics, per-type visual update, renderer submission |
| `cg_marks.c` | Persistent wall decals (bullet holes, burns, blood) and a particle simulation system |
| `cg_particles.c` | Software particle system (8192-particle pool); weather, combat, environmental effects as raw polys |
| `cg_servercmds.c` | Reliable server command handler; config strings, score/team parsing, voice chat, map restart |
| `cg_consolecmds.c` | Client console command registration and dispatch; scoreboard, weapon cycle, team orders |
| `cg_scoreboard.c` | Scoreboard overlay rendering; FFA, team, tournament, and spectator layouts with fade animation |
| `cg_info.c` | Loading screen; level screenshot, player/item icon accumulation, server metadata display |
| `cg_newdraw.c` | MissionPack (Team Arena) owner-draw HUD elements; team overlays, medals, spectator ticker |
| `tr_types.h` | Shared renderer interface types; `refEntity_t`, `refdef_t`, `glconfig_t`, `RF_*`/`RDF_*` flags |

## Core Responsibilities

- **Snapshot consumption:** Advance the simulation clock each frame by transitioning between double-buffered `snapshot_t` deliveries, detecting teleports, and firing entity/playerstate events at transition boundaries.
- **Client-side prediction:** Run `Pmove` on unacknowledged user commands to produce `cg.predictedPlayerState`, smoothing over network latency; detect and decay server-vs-client divergence errors.
- **Entity presentation:** Resolve interpolated/extrapolated positions for all `centity_t` objects each frame and dispatch type-specific rendering (players, missiles, movers, items, brushes) to the renderer.
- **Player rendering:** Load and cache 3-part player models with animations; evaluate skeletal lerp frames per frame; attach powerup, flag, shadow, and trail effects.
- **Event processing:** Translate server-generated `EV_*` entity events and playerstate-embedded events into audio, visual, and HUD feedback (obituaries, pickup notices, weapon impacts, movement sounds).
- **2D HUD composition:** Draw all screen-space UI elements each frame over the 3D view: status bar, crosshair, team overlays, center prints, scoreboards, lagometer, and (MissionPack) owner-draw elements.
- **Local entity and effect simulation:** Maintain fixed-size pools for local entities, mark decals, and particles; simulate physics, fade, and expiry; submit survivors to the renderer each frame.
- **Asset precaching:** During level init, register all required models, shaders, and sounds for weapons, items, media, and player models before gameplay begins.

## Key Interfaces & Data Flow

**Exposed to the engine:**
- `vmMain(int command, ...)` — sole cgame VM entry point; dispatches `CG_INIT`, `CG_SHUTDOWN`, `CG_DRAW_ACTIVE_FRAME`, `CG_CONSOLE_COMMAND`, `CG_CROSSHAIR_PLAYER`, `CG_LAST_ATTACKER`, and `CG_KEY_EVENT`/`CG_MOUSE_EVENT` (MissionPack).
- `CG_ConsoleCommand()` — called by the engine when a console command is not recognized as an engine command.

**Consumed from the engine (via `trap_*` syscalls, opcodes in `cg_public.h`):**
- **Snapshot/game state:** `trap_GetSnapshot`, `trap_GetCurrentSnapshotNumber`, `trap_GetGameState`, `trap_GetConfigString`
- **Renderer:** `trap_R_RegisterModel/Shader/Skin/Font`, `trap_R_AddRefEntityToScene`, `trap_R_AddPolyToScene`, `trap_R_AddLightToScene`, `trap_R_RenderScene`, `trap_R_DrawStretchPic`, `trap_R_SetColor`, `trap_R_LerpTag`
- **Sound:** `trap_S_StartSound`, `trap_S_StartLocalSound`, `trap_S_AddLoopingSound`, `trap_S_UpdateEntityPosition`, `trap_S_RegisterSound`
- **Collision:** `trap_CM_LoadMap`, `trap_CM_BoxTrace`, `trap_CM_TransformedBoxTrace`, `trap_CM_PointContents`, `trap_CM_MarkFragments`
- **Input/user commands:** `trap_GetUserCmd`, `trap_GetCurrentCmdNumber`
- **Cvar/console:** `trap_Cvar_Register/Update/Set/VariableStringBuffer`, `trap_Argv`, `trap_SendConsoleCommand`, `trap_AddCommand`

**Consumed from shared game code (`bg_*.c`):**
- `Pmove()`, `BG_EvaluateTrajectory/Delta`, `BG_PlayerStateToEntityState`, `BG_FindItemForPowerup/Holdable`, `BG_CanItemBeGrabbed`, `BG_TouchJumpPad`, `bg_itemlist[]`

**Consumed from `q_shared.c` / `q_math.c`:**
- Math: `AnglesToAxis`, `VectorMA/Copy/Normalize/Clear`, `MatrixMultiply`, `PerpendicularVector`, `CrossProduct`, `LerpAngle`
- String: `Q_strncpyz`, `Q_stricmp`, `Info_ValueForKey`, `Com_sprintf`, `va`, `COM_Parse`

## Runtime Role

**Init (`CG_INIT` → `vmMain`):**
1. `cg_main.c`: Clears all global state (`cg`, `cgs`, `cg_entities`, etc.), registers all cvars.
2. Parses `gameState` config strings for server info, item/model lists, and client slots.
3. Sequentially precaches sounds (`CG_RegisterSounds`), graphics (`CG_RegisterGraphics`), client models (`CG_NewClientInfo` × N), and weapons/items (`CG_RegisterWeapon`, `CG_RegisterItemVisuals`).
4. `cg_info.c` drives the loading screen with `trap_UpdateScreen` calls between registration steps.
5. (MissionPack) Loads script-driven HUD menus via `displayContextDef_t`/`Init_Display`.

**Frame (`CG_DRAW_ACTIVE_FRAME` → `vmMain`):**
1. `cg_snapshot.c`: Advance snapshot pipeline; fire entity and playerstate transition events.
2. `cg_servercmds.c`: Execute any newly received reliable server commands.
3. `cg_predict.c`: Run `Pmove` prediction on unacknowledged commands; update `cg.predictedPlayerState`.
4. `cg_view.c`: Compute view origin/angles/FOV; call `CG_AddPacketEntities` (`cg_ents.c`), `CG_AddViewWeapon` (`cg_weapons.c`), local entity/mark/particle add passes; submit `refdef_t` to renderer via `trap_R_RenderScene`.
5. `cg_draw.c` (`CG_DrawActive` → `CG_Draw2D`): Composite all HUD elements over the rendered frame.
6. `cg_view.c`: Drain buffered voice/announcer sounds.

**Shutdown (`CG_SHUTDOWN` → `vmMain`):**
- Not inferable in detail from provided docs beyond the `CG_SHUTDOWN` dispatch existing in `vmMain`.

## Notable Implementation Details

- **QVM / DLL duality:** All engine access is funneled through `cg_syscalls.c`'s integer-opcode `syscall()` dispatcher. Floating-point arguments are bit-cast to `int` width via `PASSFLOAT` to satisfy the integer-only ABI; the same source compiles to both QVM bytecode and native DLL.
- **Fixed-size pools throughout:** Local entities (512 slots, `cg_localents.c`), mark polys (fixed pool, `cg_marks.c`), and particles (8192 slots, `cg_particles.c`) all use doubly-linked active lists with singly-linked free lists. When the local entity pool is exhausted, the oldest active entry is evicted rather than dropping the allocation.
- **640×480 virtual coordinate system:** All 2D HUD coordinates are authored in a 640×480 space; `cg_drawtools.c` scales every draw call by `cgs.screenXScale`/`screenYScale` to the actual display resolution, isolating all rendering code from resolution changes.
- **`MAX_DLIGHTS 32` and `MAX_ENTITIES 1023` are hard renderer limits** (`tr_types.h`): the dlight limit follows from a 32-bit surface influence bitmask; the entity limit is constrained by bit packing in the renderer's drawsurf sort key.
- **Deferred client loading:** `cg_players.c` supports deferred loading of `clientInfo_t` to avoid mid-game hitches; `cg_servercmds.c` and `cg_scoreboard.c` call `CG_LoadDeferredPlayers` at safe points (scoreboard display, map restart).
- **Prediction error smoothing:** `cg_predict.c` does not snap the view on server correction; it decays the positional delta over time to avoid visible pops during normal play.
- **MissionPack feature gating:** Significant portions of `cg_consolecmds.c`, `cg_draw.c`, `cg_event.c`, `cg_servercmds.c`, and all of `cg_newdraw.c` are conditionally compiled under `#ifdef MISSIONPACK`, keeping the base Q3A and Team Arena builds from a single shared source tree.
- **Shotgun client-side seed replication:** `cg_weapons.c` reproduces the server's shotgun pellet spread RNG using the same seed transmitted in the event, so impact decals and sounds are spawned at the same positions the server computed — without any additional network data.
