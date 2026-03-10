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

## Key Types / Data Structures
| Name | Kind | Purpose |
|---|---|---|
| `playerState_t` | struct (defined elsewhere) | Full authoritative player state from server snapshot |
| `centity_t` | struct | Client-side entity; used here to route playerstate events through the entity event system |
| `cg_t` | struct | Global cgame frame state; holds damage, ammo, reward, duck, teleport fields written here |
| `cgs_t` | struct | Static cgame state; holds media handles and game rules read here |

## Global / File-Static State
| Name | Type | Scope | Purpose |
|---|---|---|---|
| `cg` | `cg_t` | global | Primary cgame state; extensively read and written |
| `cgs` | `cgs_t` | global | Static cgame config/media; read-only in this file |
| `cg_entities[]` | `centity_t[]` | global | Entity array; indexed by `clientNum` for external-event routing |

## Key Functions / Methods

### CG_CheckAmmo
- **Signature:** `void CG_CheckAmmo( void )`
- **Purpose:** Estimates seconds of ammo remaining across all held weapons; sets `cg.lowAmmoWarning` (0=ok, 1=low, 2=empty) and plays `noAmmoSound` on transitions.
- **Inputs:** `cg.snap->ps.stats[STAT_WEAPONS]`, `cg.snap->ps.ammo[]`
- **Outputs/Return:** void; mutates `cg.lowAmmoWarning`
- **Side effects:** May call `trap_S_StartLocalSound`
- **Calls:** `trap_S_StartLocalSound`
- **Notes:** Slow weapons (rocket, grenade, rail, shotgun) weight ammo at ×1000 ms; others at ×200 ms. Threshold is 5000 ms.

### CG_DamageFeedback
- **Signature:** `void CG_DamageFeedback( int yawByte, int pitchByte, int damage )`
- **Purpose:** Converts encoded damage direction and magnitude into view-kick angles (`v_dmg_pitch`, `v_dmg_roll`) and HUD indicator position (`damageX`, `damageY`).
- **Inputs:** Packed yaw/pitch bytes (255/255 = centered), raw damage count, `cg.snap->ps.stats[STAT_HEALTH]`, `cg.refdef.viewaxis`
- **Outputs/Return:** void; writes `cg.damageX/Y/Value`, `cg.v_dmg_pitch/roll/time`, `cg.attackerTime`
- **Side effects:** Mutates multiple `cg` view-kick fields
- **Calls:** `AngleVectors`, `VectorSubtract`, `DotProduct`, `VectorLength`
- **Notes:** Kick is scaled inversely with health below 40 HP; clamped to [5, 10] before view-kick math.

### CG_Respawn
- **Signature:** `void CG_Respawn( void )`
- **Purpose:** Resets client state after a respawn: suppresses movement error decay, re-syncs weapon selection, and opens weapon select UI briefly.
- **Inputs:** `cg.snap->ps.weapon`, `cg.time`
- **Outputs/Return:** void; writes `cg.thisFrameTeleport`, `cg.weaponSelectTime`, `cg.weaponSelect`
- **Side effects:** None beyond `cg` field writes
- **Calls:** None

### CG_CheckPlayerstateEvents
- **Signature:** `void CG_CheckPlayerstateEvents( playerState_t *ps, playerState_t *ops )`
- **Purpose:** Iterates the circular event ring in `ps`; fires any new or server-corrected events through `CG_EntityEvent` and records them in `cg.predictableEvents`.
- **Inputs:** Current (`ps`) and previous (`ops`) playerState event sequences
- **Outputs/Return:** void; mutates `cg.predictableEvents`, `cg.eventSequence`, `cent->currentState`
- **Side effects:** Calls `CG_EntityEvent` which may spawn effects, sounds, etc.
- **Calls:** `CG_EntityEvent`
- **Notes:** Also handles `externalEvent` (non-predicted server-pushed events) separately.

### CG_CheckChangedPredictableEvents
- **Signature:** `void CG_CheckChangedPredictableEvents( playerState_t *ps )`
- **Purpose:** After prediction, detects events that the server corrected versus what the client predicted, and re-fires the authoritative event.
- **Inputs:** Authoritative `ps`, `cg.predictableEvents[]`, `cg.eventSequence`
- **Outputs/Return:** void; may update `cg.predictableEvents`
- **Side effects:** Calls `CG_EntityEvent`; prints warning if `cg_showmiss` is set
- **Calls:** `CG_EntityEvent`, `CG_Printf`

### CG_CheckLocalSounds
- **Signature:** `void CG_CheckLocalSounds( playerState_t *ps, playerState_t *ops )`
- **Purpose:** Compares persistent stats between snapshots and plays appropriate announcer/local sounds for hits, pain, rewards, flag pickup, lead changes, timelimit, and fraglimit warnings.
- **Inputs:** `ps`, `ops`, `cg.time`, `cgs.*` (timelimit, fraglimit, scores, media handles)
- **Outputs/Return:** void; mutates `cg.timelimitWarnings`, `cg.fraglimitWarnings`, reward stack
- **Side effects:** Calls `trap_S_StartLocalSound`, `CG_AddBufferedSound`, `pushReward`, `CG_PainEvent`
- **Calls:** `trap_S_StartLocalSound`, `CG_AddBufferedSound`, `CG_PainEvent`, `pushReward`
- **Notes:** Skipped entirely during intermission or when spectating. Reward stack is bounded by `MAX_REWARDSTACK`.

### CG_TransitionPlayerState
- **Signature:** `void CG_TransitionPlayerState( playerState_t *ps, playerState_t *ops )`
- **Purpose:** Top-level orchestrator called each frame; detects follow-mode switches, damage events, respawns, map restarts, duck transitions, and delegates to all sub-functions.
- **Inputs:** Current and previous playerState pointers
- **Outputs/Return:** void
- **Side effects:** May overwrite `*ops`; sets `cg.thisFrameTeleport`, `cg.duckChange/Time`, `cg.mapRestart`
- **Calls:** `CG_DamageFeedback`, `CG_Respawn`, `CG_CheckLocalSounds`, `CG_CheckAmmo`, `CG_CheckPlayerstateEvents`

### pushReward *(static)*
- Pushes a (sound, shader, count) triple onto `cg.rewardStack` if capacity remains.

## Control Flow Notes
`CG_TransitionPlayerState` is the primary entry point, called from `cg_snapshot.c` / `cg_predict.c` each frame after snapshot processing or prediction. `CG_CheckChangedPredictableEvents` is called separately from `cg_predict.c` after the prediction loop to reconcile mispredictions. Neither function participates in rendering; they feed state that draw code reads later in the same frame.

## External Dependencies
- `cg_local.h` → pulls in `q_shared.h`, `bg_public.h`, `cg_public.h`, `tr_types.h`
- **Defined elsewhere:** `CG_EntityEvent`, `CG_PainEvent`, `CG_AddBufferedSound`, `AngleVectors`, `DotProduct`, `VectorSubtract`, `VectorLength`, `trap_S_StartLocalSound`, `cg`, `cgs`, `cg_entities`, `cg_showmiss`
