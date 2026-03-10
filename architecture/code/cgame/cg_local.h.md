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

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `lerpFrame_t` | struct | Interpolation state for a single model part (frame, time, yaw/pitch, animation pointer) |
| `playerEntity_t` | struct | Per-player extra render state: leg/torso/flag lerp frames, railgun flash, barrel spin |
| `centity_t` | struct | Client-side entity mirroring `gentity_t`; holds current/next snapshot states, interpolated origin/angles, player entity data |
| `markPoly_t` | struct | Decal/impact mark polygon in a doubly-linked list with shader, color, fade, and geometry |
| `localEntity_t` | struct | Purely client-side transient effect entity (explosion, fragment, smoke, score plum, etc.) with trajectory and lifetime |
| `score_t` | struct | Per-client scoreboard entry (score, ping, powerups, awards) |
| `clientInfo_t` | struct | All media and metadata for one connected client: model handles, skin handles, sounds, animations, team/handicap info |
| `weaponInfo_t` | struct | Registered media for one weapon: models, sounds, missile trail/brass function pointers, dlight info |
| `itemInfo_t` | struct | Registered media for one item (models and icon handle) |
| `skulltrail_t` | struct | Ring buffer of skull positions for skull-trail effect (MissionPack) |
| `cg_t` | struct | Per-frame volatile cgame state: time, snapshot pointers, prediction state, view, HUD timers, score data, sound buffers |
| `cgMedia_t` | struct | All pre-registered renderer and sound handles loaded at level start |
| `cgs_t` | struct | Static cgame state persisting across tournament restarts: gamestate, parsed serverinfo, client infos, team chat |
| `leType_t` | enum | Local entity type tag (mark, explosion, fragment, fade variants, kamikaze, etc.) |
| `leFlag_t` | enum | Bit flags for local entity behavior (no-scale, tumble, kamikaze sounds) |
| `leMarkType_t` | enum | What wall mark a fragment leaves (none, burn, blood) |
| `leBounceSoundType_t` | enum | What sound a bouncing fragment makes (none, blood, brass) |
| `footstep_t` | enum | Footstep material categories for sound selection |
| `impactSound_t` | enum | Weapon impact surface categories (default, metal, flesh) |
| `q3print_t` | enum | Print channel types for console/chat/teamchat |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cgs` | `cgs_t` | global (extern) | Static cgame state; gamestate, serverinfo, client array, media |
| `cg` | `cg_t` | global (extern) | Live per-frame cgame state; time, snapshots, prediction, HUD |
| `cg_entities[MAX_GENTITIES]` | `centity_t[]` | global (extern) | Client-side entity array parallel to server's gentity array |
| `cg_weapons[MAX_WEAPONS]` | `weaponInfo_t[]` | global (extern) | Registered weapon media cache |
| `cg_items[MAX_ITEMS]` | `itemInfo_t[]` | global (extern) | Registered item media cache |
| `cg_markPolys[MAX_MARK_POLYS]` | `markPoly_t[]` | global (extern) | Pool of wall-mark decal polygons |
| `initparticles` | `qboolean` | global (extern) | Flag indicating particle system has been initialized |
| `sortedTeamPlayers[TEAM_MAXOVERLAY]` | `int[]` | global (extern, in cg_draw.c) | Sorted client indices for team overlay |
| `numSortedTeamPlayers` | `int` | global (extern) | Count of valid entries in above array |
| `drawTeamOverlayModificationCount` | `int` | global (extern) | Change counter to detect team overlay staleness |
| `systemChat[256]`, `teamChat1[256]`, `teamChat2[256]` | `char[]` | global (extern) | Chat line display buffers |
| ~80 `vmCvar_t` externs | `vmCvar_t` | global (extern) | All user-facing cvars for cgame (fov, crosshair, gun position, shadows, etc.) |

## Key Functions / Methods

### CG_DrawActiveFrame
- Signature: `void CG_DrawActiveFrame( int serverTime, stereoFrame_t stereoView, qboolean demoPlayback )`
- Purpose: Main per-frame entry point; builds and renders the 3D scene plus HUD for one client frame.
- Inputs: Server time in msec, stereo eye selection, demo playback flag.
- Outputs/Return: None; submits render commands to the engine.
- Side effects: Updates `cg.time`, advances entity states, calls prediction, adds all scene entities and polys.
- Calls: Virtually every cgame subsystem (predict, ents, weapons, effects, marks, draw).
- Notes: Only called from `CG_DRAW_ACTIVE_FRAME` export dispatch.

### CG_PredictPlayerState
- Signature: `void CG_PredictPlayerState( void )`
- Purpose: Client-side movement prediction; replays un-acknowledged user commands against pmove to reduce perceived latency.
- Inputs: None (reads `cg`, `cg_entities`, user command ring).
- Outputs/Return: Writes `cg.predictedPlayerState`.
- Side effects: May fire `CG_PainEvent`; sets `cg.validPPS`.
- Calls: `Pmove`, `CG_Trace`, `CG_PointContents`, `CG_CheckChangedPredictableEvents`.

### CG_ProcessSnapshots
- Signature: `void CG_ProcessSnapshots( void )`
- Purpose: Advances `cg.snap`/`cg.nextSnap` by fetching new snapshots from the engine and processing server commands.
- Inputs: None.
- Outputs/Return: None; updates `cg.snap`, `cg.nextSnap`, `cg.time`.
- Side effects: Calls `CG_ExecuteNewServerCommands`, triggers cgame state transitions.
- Calls: `trap_GetCurrentSnapshotNumber`, `trap_GetSnapshot`, `trap_GetServerCommand`.

### CG_AddPacketEntities
- Signature: `void CG_AddPacketEntities( void )`
- Purpose: Iterates all entities in the current snapshot and adds render entities/effects to the scene.
- Inputs: None (reads `cg.snap`).
- Side effects: Calls per-entity dispatch (player, item, missile, mover, beam, etc.); may spawn local entities.

### CG_Player
- Signature: `void CG_Player( centity_t *cent )`
- Purpose: Renders a player entity: resolves animations for legs/torso, attaches weapon, applies powerup effects.
- Inputs: `cent` — the client entity to render.
- Side effects: Calls `trap_R_AddRefEntityToScene` multiple times; may add dlight and shadow.

### trap_* functions (system traps)
- These are not defined here; they are stubs resolved by the VM syscall dispatch at runtime.
- Cover: console I/O, cvar access, filesystem, network commands, collision (`CM_*`), sound (`S_*`), renderer (`R_*`), snapshot access, cinematic playback, key state.
- Notes: The entire cgame VM boundary is expressed through these ~80 trap prototypes.

## Control Flow Notes
- The cgame module is loaded fresh on each level change; `CG_Init` (exported via `CG_INIT`) registers all media and initializes `cg`/`cgs`.
- Each render frame the engine calls `CG_DrawActiveFrame` → `CG_ProcessSnapshots` → `CG_PredictPlayerState` → entity/effect update → scene submission → 2D HUD draw.
- `cgs_t` survives tournament restarts; `cg_t` is reset each frame cycle.
- The module is self-contained: no direct C linkage to engine internals; all communication is via `trap_*` calls.

## External Dependencies
- `../game/q_shared.h` — shared math, string, entity/player state, cvar, trace types
- `tr_types.h` — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t`, etc.)
- `../game/bg_public.h` — gameplay constants, animation enums, `gitem_t`, `pmove_t`, `playerState_t`
- `cg_public.h` — `snapshot_t`, `cgameImport_t`/`cgameExport_t` enums, import API version
- All `trap_*` symbols: defined in engine executable, resolved at VM load time (not in this file)
- `gitem_t bg_itemlist[]`, `Pmove()`: defined in `bg_*.c` shared game code
