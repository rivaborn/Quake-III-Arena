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

## Key Types / Data Structures
| Name | Kind | Purpose |
|------|------|---------|
| `clientInfo_t` | struct (defined in `cg_local.h`) | All per-client media references, animation data, sounds, and metadata |
| `lerpFrame_t` | struct (defined in `cg_local.h`) | Animation interpolation state for one body part (frame, oldFrame, backlerp, yaw/pitch swing) |
| `playerEntity_t` | struct (defined in `cg_local.h`) | Groups lerpFrame_t for legs, torso, flag; tracks pain, barrel spin, rail flash |
| `centity_t` | struct (defined in `cg_local.h`) | Client-side entity; carries entityState, playerEntity_t, lerp origin/angles |
| `animation_t` | struct (defined in `bg_public.h`) | Single animation clip: firstFrame, numFrames, loopFrames, frameLerp, reversed, flipflop |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `cg_customSoundNames` | `char *[MAX_CUSTOM_SOUNDS]` | global | Maps custom sound indices to wildcard filenames (e.g., `*death1.wav`) |

## Key Functions / Methods

### CG_CustomSound
- Signature: `sfxHandle_t CG_CustomSound( int clientNum, const char *soundName )`
- Purpose: Resolve a sound name to a registered sfxHandle. Non-wildcard names go through direct registration; wildcard names (`*`) look up the client's preloaded sound array.
- Inputs: client index, sound name string
- Outputs/Return: `sfxHandle_t` handle
- Side effects: Calls `trap_S_RegisterSound` for non-custom sounds
- Calls: `trap_S_RegisterSound`, `CG_Error`

### CG_ParseAnimationFile
- Signature: `static qboolean CG_ParseAnimationFile( const char *filename, clientInfo_t *ci )`
- Purpose: Read and parse an `animation.cfg` file, populating `ci->animations[]`, footstep type, head offset, and gender. Constructs synthetic backward/flag animations after parsing.
- Inputs: path to config file, target `clientInfo_t`
- Outputs/Return: `qtrue` on success
- Side effects: Fills `ci->animations[MAX_TOTALANIMATIONS]`, `ci->footsteps`, `ci->headOffset`, `ci->gender`, `ci->fixedlegs/fixedtorso`. Reads from filesystem.
- Calls: `trap_FS_FOpenFile`, `trap_FS_Read`, `trap_FS_FCloseFile`, `COM_Parse`, `CG_Printf`
- Notes: Adjusts leg-only frame indices by subtracting the `TORSO_GESTURE` base. Missing Team Arena animations (`TORSO_GETFLAG`–`TORSO_NEGATIVE`) are aliased to `TORSO_GESTURE`.

### CG_RegisterClientSkin
- Signature: `static qboolean CG_RegisterClientSkin( clientInfo_t *ci, ... )`
- Purpose: Locate and register skin files for legs, torso, and head using team/gametype-aware path resolution.
- Inputs: `clientInfo_t`, team name, model name, skin name, head model/skin names
- Outputs/Return: `qtrue` if all three skins loaded successfully
- Side effects: Sets `ci->legsSkin`, `ci->torsoSkin`, `ci->headSkin`
- Calls: `CG_FindClientModelFile`, `CG_FindClientHeadFile`, `trap_R_RegisterSkin`

### CG_RegisterClientModelname
- Signature: `static qboolean CG_RegisterClientModelname( clientInfo_t *ci, ... )`
- Purpose: Load all three MD3 model files, register skins, parse the animation config, and register the model icon shader.
- Inputs: `clientInfo_t`, model/skin/head model/head skin/team name strings
- Outputs/Return: `qtrue` on complete success
- Side effects: Sets `ci->legsModel`, `ci->torsoModel`, `ci->headModel`, `ci->modelIcon`; populates animations via `CG_ParseAnimationFile`
- Calls: `trap_R_RegisterModel`, `CG_RegisterClientSkin`, `CG_ParseAnimationFile`, `trap_R_RegisterShaderNoMip`, `CG_FindClientHeadFile`

### CG_LoadClientInfo
- Signature: `static void CG_LoadClientInfo( clientInfo_t *ci )`
- Purpose: Top-level media loader for a client: calls `CG_RegisterClientModelname` with fallback to defaults, detects new-animation rigs via `tag_flag`, registers custom sounds, and resets any existing player entities using this client.
- Inputs: `clientInfo_t *ci`
- Side effects: Modifies `ci->newAnims`, `ci->sounds[]`, `ci->deferred`; calls `CG_ResetPlayerEntity` for all matching entities; triggers renderer/sound registration
- Calls: `CG_RegisterClientModelname`, `trap_R_LerpTag`, `trap_S_RegisterSound`, `CG_ResetPlayerEntity`, `CG_Error`

### CG_NewClientInfo
- Signature: `void CG_NewClientInfo( int clientNum )`
- Purpose: Called when a client's configstring changes. Parses all player attributes (name, color, model, head model, team, etc.) into a temp `clientInfo_t`, then either scans for an existing match to copy from, defers the load, or loads immediately.
- Inputs: client index
- Side effects: Overwrites `cgs.clientinfo[clientNum]`; may trigger deferred or immediate model load; reads cvars
- Calls: `CG_ConfigString`, `Info_ValueForKey`, `CG_ColorFromString`, `CG_ScanForExistingClientInfo`, `CG_SetDeferredClientInfo`, `CG_LoadClientInfo`, `trap_MemoryRemaining`, `trap_Cvar_VariableStringBuffer`

### CG_LoadDeferredPlayers
- Signature: `void CG_LoadDeferredPlayers( void )`
- Purpose: Called each frame while the scoreboard is visible to opportunistically load any pending deferred client infos, skipping if memory is critically low.
- Side effects: Calls `CG_LoadClientInfo` per deferred entry

### CG_RunLerpFrame
- Signature: `static void CG_RunLerpFrame( clientInfo_t *ci, lerpFrame_t *lf, int newAnimation, float speedScale )`
- Purpose: Advance the animation lerp state for one body part based on `cg.time`. Handles looping, reversed, and flipflop animations; computes `backlerp`.
- Inputs: client info, lerp frame state, new animation index, speed scale (haste, etc.)
- Side effects: Mutates `lf->frame`, `lf->oldFrame`, `lf->frameTime`, `lf->backlerp`
- Calls: `CG_SetLerpFrameAnimation`

### CG_PlayerAngles
- Signature: `static void CG_PlayerAngles( centity_t *cent, vec3_t legs[3], vec3_t torso[3], vec3_t head[3] )`
- Purpose: Compute orientation matrices for all three body segments each frame. Handles yaw swing, pitch fraction, velocity roll lean, `fixedlegs`/`fixedtorso` flags, and pain twitch.
- Side effects: Mutates `cent->pe.torso` and `cent->pe.legs` yaw/pitch state
- Calls: `CG_SwingAngles`, `CG_AddPainTwitch`, `AnglesToAxis`, `AnglesSubtract`, `AngleSubtract`, `AngleMod`, `VectorNormalize`, `DotProduct`

### CG_Player
- Signature: `void CG_Player( centity_t *cent )`
- Purpose: Main per-frame player rendering entry point. Assembles legs/torso/head refEntities, applies powerup overlays, submits shadow, splash, sprites, weapon, and all optional MISSIONPACK effects to the scene.
- Side effects: Multiple `trap_R_AddRefEntityToScene` calls; may add lights, sounds, and local entities
- Calls: `CG_PlayerAngles`, `CG_PlayerAnimation`, `CG_PlayerSprites`, `CG_PlayerShadow`, `CG_PlayerSplash`, `CG_AddRefEntityWithPowerups`, `CG_PositionRotatedEntityOnTag`, `CG_AddPlayerWeapon`, `CG_PlayerPowerups`, `CG_PlayerTokens` (MISSIONPACK), `CG_BreathPuffs` (MISSIONPACK), `CG_DustTrail` (MISSIONPACK)

### CG_ResetPlayerEntity
- Signature: `void CG_ResetPlayerEntity( centity_t *cent )`
- Purpose: Reset all animation and positional state when a player teleports or first enters view.
- Side effects: Clears `cent->pe.legs/torso`, resets lerp frames via `CG_ClearLerpFrame`, evaluates trajectory to set `lerpOrigin/lerpAngles`
- Calls: `CG_ClearLerpFrame`, `BG_EvaluateTrajectory`

### CG_AddRefEntityWithPowerups
- Signature: `void CG_AddRefEntityWithPowerups( refEntity_t *ent, entityState_t *state, int team )`
- Purpose: Submit a refEntity to the scene, applying powerup shader overlays (quad, regen, battlesuit, invisibility) as additional draw passes. Also used by missile rendering.
- Side effects: One or more `trap_R_AddRefEntityToScene` calls; modifies `ent->customShader`

## Control Flow Notes
- `CG_NewClientInfo` is the init path, triggered by configstring changes (level load or player join/change).
- `CG_LoadDeferredPlayers` runs each frame during scoreboard display to drain the deferred queue.
- `CG_Player` is the per-frame render path, called from `cg_ents.c:CG_AddPacketEntities` for every `ET_PLAYER` entity visible in the current snapshot.
- Animation state (`CG_RunLerpFrame`) is updated inside `CG_Player` → `CG_PlayerAnimation` every frame.

## External Dependencies
- `cg_local.h` → `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- `COM_Parse`, `Info_ValueForKey`, `Q_stricmp`, `Q_strncpyz`, `VectorCopy/Clear/Set/MA/Normalize`, `DotProduct`, `AnglesToAxis`, `AngleMod`, `BG_EvaluateTrajectory` — defined in shared/game code
- `trap_*` functions — VM syscall stubs defined in `cg_syscalls.c`
- `CG_SmokePuff`, `CG_ImpactMark`, `CG_PositionRotatedEntityOnTag`, `CG_PositionEntityOnTag`, `CG_AddPlayerWeapon` — defined in other cgame modules
- `cgs`, `cg`, `cg_entities` — global state defined in `cg_main.c`
