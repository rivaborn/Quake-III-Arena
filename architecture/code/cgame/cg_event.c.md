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

## Key Types / Data Structures
None defined in this file; relies on types from `cg_local.h`.

| Name | Kind | Purpose |
|---|---|---|
| `centity_t` | struct (extern) | Client entity with event tracking fields (`previousEvent`, `lerpOrigin`, `pe`) |
| `entityState_t` | struct (extern) | Snapshot-transmitted entity state; carries `event`, `eventParm`, `weapon`, etc. |
| `clientInfo_t` | struct (extern) | Per-client media/gender/footstep info used for obituary and sound selection |
| `localEntity_t` | struct (extern) | Spawned by `CG_SmokePuff` for jump pad visual effect |

## Global / File-Static State
None declared in this file. All state accessed via `cg` and `cgs` extern globals from `cg_local.h`.

## Key Functions / Methods

### CG_PlaceString
- **Signature:** `const char *CG_PlaceString( int rank )`
- **Purpose:** Converts a numeric rank to an ordinal string (e.g., "1st", "2nd"), with color coding for top 3 and optional "Tied for" prefix.
- **Inputs:** `rank` — integer rank, may have `RANK_TIED_FLAG` bit set.
- **Outputs/Return:** Pointer to internal static `str[64]`; not re-entrant.
- **Side effects:** Writes to a file-scoped static buffer.
- **Calls:** `Com_sprintf`, `va`
- **Notes:** Called externally by scoreboard drawing code.

---

### CG_Obituary
- **Signature:** `static void CG_Obituary( entityState_t *ent )`
- **Purpose:** Prints a death message to the console and, if the local player made the kill, a center-print frag notification with current rank/score.
- **Inputs:** `ent` — entity state carrying `otherEntityNum` (target), `otherEntityNum2` (attacker), `eventParm` (MOD_* constant).
- **Outputs/Return:** void
- **Side effects:** Writes to `cg.killerName` when local player is killed; calls `CG_Printf`, `CG_CenterPrint`.
- **Calls:** `CG_Error`, `CG_ConfigString`, `Info_ValueForKey`, `Q_strncpyz`, `strcat`, `CG_PlaceString`, `CG_CenterPrint`, `CG_Printf`
- **Notes:** Gender-aware self-kill messages; MissionPack-only MODs gated by `#ifdef MISSIONPACK`.

---

### CG_UseItem
- **Signature:** `static void CG_UseItem( centity_t *cent )`
- **Purpose:** Processes a holdable item use event: prints center-print for local player, updates `medkitUsageTime`, and plays appropriate sound.
- **Inputs:** `cent` — entity triggering the use event; item index derived from `event & ~EV_EVENT_BITS`.
- **Outputs/Return:** void
- **Side effects:** Mutates `cgs.clientinfo[clientNum].medkitUsageTime`; calls `trap_S_StartSound`.
- **Calls:** `BG_FindItemForHoldable`, `CG_CenterPrint`, `trap_S_StartSound`

---

### CG_ItemPickup
- **Signature:** `static void CG_ItemPickup( int itemNum )`
- **Purpose:** Records a new item pickup on the local player's HUD state and triggers auto-weapon switch for weapons if `cg_autoswitch` is set.
- **Inputs:** `itemNum` — index into `bg_itemlist`.
- **Outputs/Return:** void
- **Side effects:** Sets `cg.itemPickup`, `cg.itemPickupTime`, `cg.itemPickupBlendTime`, `cg.weaponSelectTime`, `cg.weaponSelect`.

---

### CG_PainEvent
- **Signature:** `void CG_PainEvent( centity_t *cent, int health )`
- **Purpose:** Plays a health-tier-appropriate pain sound for an entity, rate-limited to one per 500ms.
- **Inputs:** `cent` — entity in pain; `health` — current health value selecting sound tier.
- **Outputs/Return:** void
- **Side effects:** Sets `cent->pe.painTime`, toggles `cent->pe.painDirection`.
- **Calls:** `trap_S_StartSound`, `CG_CustomSound`

---

### CG_EntityEvent
- **Signature:** `void CG_EntityEvent( centity_t *cent, vec3_t position )`
- **Purpose:** Central event dispatcher: reads `event` from entity state and routes to sound, visual, or game-logic handlers via a large `switch` on `EV_*` constants.
- **Inputs:** `cent` — entity with pending event; `position` — world-space position for positional audio/effects.
- **Outputs/Return:** void
- **Side effects:** Calls numerous subsystems (sound, effects, weapons, HUD). Mutates `cg` global fields (`landChange`, `landTime`, `stepChange`, `stepTime`, `powerupActive`, `powerupTime`).
- **Calls:** `CG_SmokePuff`, `CG_FireWeapon`, `CG_UseItem`, `CG_ItemPickup`, `CG_PainEvent`, `CG_Obituary`, `CG_SpawnEffect`, `CG_MissileHitPlayer`, `CG_MissileHitWall`, `CG_RailTrail`, `CG_Bullet`, `CG_ShotgunFire`, `CG_ScorePlum`, `CG_GibPlayer`, `CG_AddBufferedSound`, `CG_VoiceChatLocal`, `CG_OutOfAmmoChange`, `CG_Beam`, `trap_S_StartSound`, `trap_S_StopLoopingSound`, `ByteToDir`, and MissionPack-specific effects.
- **Notes:** `DEBUGNAME` macro emits event name to console when `cg_debugEvents` is set. Unknown events call `CG_Error`.

---

### CG_CheckEvents
- **Signature:** `void CG_CheckEvents( centity_t *cent )`
- **Purpose:** Entry point called per-entity per-frame; determines whether a new event is pending by comparing `currentState.event` to `previousEvent`, then invokes `CG_EntityEvent`.
- **Inputs:** `cent` — any client entity.
- **Outputs/Return:** void
- **Side effects:** Mutates `cent->previousEvent`; calls `BG_EvaluateTrajectory`, `CG_SetEntitySoundPosition`, `CG_EntityEvent`.
- **Notes:** Event-only entities (`eType > ET_EVENTS`) fire exactly once. `EF_PLAYER_EVENT` remaps entity number to the owning client before dispatch.

## Control Flow Notes
- Called during the frame update pipeline from `CG_AddPacketEntities` (in `cg_ents.c`) per entity, and from `CG_CheckPlayerstateEvents` (in `cg_playerstate.c`) for predicted player events.
- No init or shutdown involvement; purely frame-driven.

## External Dependencies
- `cg_local.h` — all cgame types, globals (`cg`, `cgs`), and trap declarations.
- `ui/menudef.h` — `VOICECHAT_*` constants (MissionPack only).
- **Defined elsewhere:** `BG_EvaluateTrajectory`, `BG_FindItemForHoldable`, `ByteToDir`, `Info_ValueForKey`, `Q_strncpyz`, `Com_sprintf`, `va`; all `CG_*` effect/weapon functions; all `trap_S_*` sound traps.
