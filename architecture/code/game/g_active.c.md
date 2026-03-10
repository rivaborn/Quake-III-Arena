# code/game/g_active.c

## File Purpose
Implements per-client per-frame logic for the server-side game module, covering player movement, environmental effects, damage feedback, event dispatch, and end-of-frame state synchronization. It is the central "think" driver for all connected clients each server frame.

## Core Responsibilities
- Run `Pmove` physics simulation for each client and propagate results back to entity state
- Apply world environmental damage (drowning, lava, slime) each frame
- Aggregate and encode damage feedback into `playerState_t` for pain blends/kicks
- Dispatch and process server-authoritative client events (falling, weapon fire, item use, teleport)
- Handle spectator movement and chase-cam follow logic
- Enforce inactivity kick timer and respawn conditions
- Execute once-per-second timer actions (health regen, armor decay, ammo regen via MISSIONPACK)
- Synchronize `playerState_t` → `entityState_t` and send predictable events to other clients

## Key Types / Data Structures
None defined here; all types are from `g_local.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `pmove_t` | struct (external) | Input/output block passed to `Pmove`; carries player state, command, trace mask, results |
| `gentity_t` | struct (external) | Server-side game entity; player entity driven by this file |
| `gclient_t` | struct (external) | Per-client server state (player state, persistent data, session, timers) |
| `level_locals_t` | struct (external) | Global level state accessed as `level` |

## Global / File-Static State
None defined in this file; all global state is declared in `g_local.h`.

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `level` | `level_locals_t` | global (extern) | Frame time, client array, sound indices, intermission flags |
| `g_entities` | `gentity_t[]` | global (extern) | All game entities |
| `pmove_fixed`, `pmove_msec`, `g_speed`, `g_gravity`, etc. | `vmCvar_t` | global (extern) | Cvars controlling movement and gameplay |

## Key Functions / Methods

### P_DamageFeedback
- **Signature:** `void P_DamageFeedback( gentity_t *player )`
- **Purpose:** Encodes accumulated frame damage into `playerState_t` fields for client-side pain rendering, and fires `EV_PAIN` sound event.
- **Inputs:** `player` — the player entity
- **Outputs/Return:** void; mutates `client->ps.damagePitch/Yaw/Count/damageEvent`
- **Side effects:** Clears `damage_blood`, `damage_armor`, `damage_knockback`; calls `G_AddEvent`
- **Calls:** `vectoangles`, `G_AddEvent`
- **Notes:** Skips dead players (`PM_DEAD`). World damage (falls, etc.) uses sentinel pitch/yaw = 255 for centered blend. Pain sound is debounced to 700 ms.

### P_WorldEffects
- **Signature:** `void P_WorldEffects( gentity_t *ent )`
- **Purpose:** Applies drowning damage and lava/slime sizzle damage based on water level and content type.
- **Inputs:** `ent` — player entity
- **Outputs/Return:** void; modifies `ent->health`, `ent->damage`, `client->airOutTime`
- **Side effects:** Calls `G_Damage`, `G_Sound`, `G_AddEvent`; noclip bypasses all effects
- **Calls:** `G_Damage`, `G_Sound`, `G_SoundIndex`, `G_AddEvent`
- **Notes:** Drowning damage ramps from 2 to 15 HP/sec. Envirosuit negates all environmental damage.

### G_SetClientSound
- **Signature:** `void G_SetClientSound( gentity_t *ent )`
- **Purpose:** Sets the looping ambient sound for a client (sizzle in lava/slime, silence otherwise).
- **Inputs/Outputs:** Mutates `client->ps.loopSound`
- **Side effects:** None beyond the field write
- **Notes:** MISSIONPACK adds prox-mine tick sound via `EF_TICKING`.

### ClientImpacts
- **Signature:** `void ClientImpacts( gentity_t *ent, pmove_t *pm )`
- **Purpose:** After `Pmove`, invokes `touch` callbacks for every entity the player contacted, deduplicating the touch list.
- **Calls:** `ent->touch`, `other->touch`
- **Notes:** Bots also fire their own `touch` callback against touched entities.

### G_TouchTriggers
- **Signature:** `void G_TouchTriggers( gentity_t *ent )`
- **Purpose:** Broad-phase + narrow-phase trigger overlap test; fires `touch` on all overlapping trigger entities.
- **Calls:** `trap_EntitiesInBox`, `trap_EntityContact`, `BG_PlayerTouchesItem`, `hit->touch`, `ent->touch`
- **Notes:** Spectators only interact with teleporters and doors. Jump-pad frame tracking is reset if no pad was touched this pmove frame.

### SpectatorThink
- **Signature:** `void SpectatorThink( gentity_t *ent, usercmd_t *ucmd )`
- **Purpose:** Runs spectator-specific movement (PM_SPECTATOR pmove) and handles attack-button cycling through players.
- **Calls:** `Pmove`, `G_TouchTriggers`, `trap_UnlinkEntity`, `Cmd_FollowCycle_f`
- **Notes:** Follow-spectators skip pmove entirely; free spectators fly through bodies (`~CONTENTS_BODY`).

### ClientInactivityTimer
- **Signature:** `qboolean ClientInactivityTimer( gclient_t *client )`
- **Purpose:** Drops clients that have not sent movement input within `g_inactivity` seconds; sends a 10-second warning.
- **Outputs/Return:** `qfalse` if the client was dropped
- **Side effects:** `trap_DropClient`, `trap_SendServerCommand`

### ClientTimerActions
- **Signature:** `void ClientTimerActions( gentity_t *ent, int msec )`
- **Purpose:** Once-per-second bookkeeping: health regen (Regen powerup / GUARD), health/armor decay above max, and MISSIONPACK per-weapon ammo regen.
- **Side effects:** Mutates `ent->health`, `client->ps.stats[STAT_ARMOR]`, `client->ps.ammo[]`; fires `EV_POWERUP_REGEN`

### ClientEvents
- **Signature:** `void ClientEvents( gentity_t *ent, int oldEventSequence )`
- **Purpose:** Iterates new player-state events since last frame and executes server-authoritative consequences (fall damage, fire weapon, teleporter use, medkit, kamikaze, etc.).
- **Calls:** `G_Damage`, `FireWeapon`, `Drop_Item`, `TeleportPlayer`, `SelectSpawnPoint`, `G_StartKamikaze`, `DropPortalSource/Destination`

### SendPendingPredictableEvents
- **Signature:** `void SendPendingPredictableEvents( playerState_t *ps )`
- **Purpose:** Creates a temporary entity replicating any unacknowledged predictable events so all *other* clients receive them.
- **Calls:** `G_TempEntity`, `BG_PlayerStateToEntityState`

### ClientThink_real
- **Signature:** `void ClientThink_real( gentity_t *ent )`
- **Purpose:** Main per-client frame driver: validates command time, runs pmove, syncs entity state, dispatches events, triggers, impacts, handles respawn and inactivity.
- **Calls:** `Pmove`, `BG_PlayerStateToEntityState(ExtraPolate)`, `ClientEvents`, `G_TouchTriggers`, `ClientImpacts`, `ClientTimerActions`, `BotTestAAS`, `SendPendingPredictableEvents`, `CheckGauntletAttack`, `Weapon_HookFree`, `ClientIntermissionThink`, `SpectatorThink`, `ClientInactivityTimer`, `respawn`, `trap_LinkEntity`
- **Notes:** Command time is clamped to ±200 ms/1000 ms of `level.time` as anti-cheat. `pmove_msec` is clamped to [8,33].

### ClientThink
- **Signature:** `void ClientThink( int clientNum )`
- **Purpose:** Engine-facing entry point called when a new user command arrives; reads the usercmd and calls `ClientThink_real` for non-bot, non-synchronous clients.
- **Calls:** `trap_GetUsercmd`, `ClientThink_real`

### G_RunClient
- **Signature:** `void G_RunClient( gentity_t *ent )`
- **Purpose:** Server-frame entry point for bots and synchronous-client mode; forces `serverTime = level.time` then calls `ClientThink_real`.

### ClientEndFrame
- **Signature:** `void ClientEndFrame( gentity_t *ent )`
- **Purpose:** End-of-frame cleanup: expire powerups, apply world effects and damage feedback, set connection flag, sync entity state, send predictable events.
- **Calls:** `SpectatorClientEndFrame`, `P_WorldEffects`, `P_DamageFeedback`, `G_SetClientSound`, `BG_PlayerStateToEntityState(ExtraPolate)`, `SendPendingPredictableEvents`

### SpectatorClientEndFrame
- **Signature:** `void SpectatorClientEndFrame( gentity_t *ent )`
- **Purpose:** For follow-spectators, copies the followed client's `playerState_t` wholesale; manages scoreboard flag.

## Control Flow Notes
- **Per-command (variable rate):** `ClientThink` → `ClientThink_real` — runs for every arriving user command (may be multiple per server frame for fast clients).
- **Per-server-frame:** `G_RunClient` (bots / synchronous mode) → `ClientThink_real`; `ClientEndFrame` is called once at end of each server frame for every client.
- Initialization and shutdown are handled elsewhere (`g_client.c`). This file owns only the frame-loop callbacks.

## External Dependencies
- `g_local.h` (pulls `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:** `Pmove` (bg_pmove.c), `G_Damage`, `G_AddEvent`, `G_Sound`, `G_TempEntity`, `G_SoundIndex` (g_utils/g_combat), `BG_PlayerStateToEntityState`, `BG_PlayerTouchesItem` (bg_misc.c), `FireWeapon`, `CheckGauntletAttack`, `Weapon_HookFree` (g_weapon.c), `TeleportPlayer`, `SelectSpawnPoint`, `respawn` (g_client/g_misc), `Drop_Item` (g_items.c), `BotTestAAS` (ai_main.c), all `trap_*` syscalls (g_syscalls.c)
