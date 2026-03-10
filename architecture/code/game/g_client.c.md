# code/game/g_client.c

## File Purpose
Manages the full client lifecycle within the game module: connection, spawning, respawning, userinfo updates, body queue management, and disconnection. Handles spawn point selection logic and player state initialization at each spawn.

## Core Responsibilities
- Spawn point registration (`SP_info_player_*`) and selection (nearest, random, furthest, initial, spectator)
- Body queue management: pooling corpse entities, animating their sink/disappearance
- Client lifecycle callbacks: `ClientConnect`, `ClientBegin`, `ClientSpawn`, `ClientDisconnect`
- Userinfo parsing and configstring broadcasting (`ClientUserinfoChanged`)
- Player name sanitization (`ClientCleanName`)
- Team utility queries: `TeamCount`, `TeamLeader`, `PickTeam`
- View angle delta computation (`SetClientViewAngle`)

## Key Types / Data Structures

None defined in this file — all types come from `g_local.h`.

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `playerMins` | `static vec3_t` | file-static | Player bounding box minimum used for telefrag and body checks |
| `playerMaxs` | `static vec3_t` | file-static | Player bounding box maximum used for telefrag and body checks |

## Key Functions / Methods

### SpotWouldTelefrag
- **Signature:** `qboolean SpotWouldTelefrag( gentity_t *spot )`
- **Purpose:** Tests whether spawning at `spot` would overlap a live client entity.
- **Inputs:** `spot` — candidate spawn point entity.
- **Outputs/Return:** `qtrue` if any client entity overlaps the spawn box.
- **Side effects:** None.
- **Calls:** `trap_EntitiesInBox`
- **Notes:** Uses `playerMins`/`playerMaxs` offset from the spot origin.

### SelectRandomFurthestSpawnPoint
- **Signature:** `gentity_t *SelectRandomFurthestSpawnPoint( vec3_t avoidPoint, vec3_t origin, vec3_t angles )`
- **Purpose:** Finds a ranked list of non-telefragging spawn points sorted by distance from `avoidPoint`, then picks randomly from the top half.
- **Inputs:** `avoidPoint` — location to stay far from (typically last death origin); output params `origin`, `angles`.
- **Outputs/Return:** Chosen spawn point entity; writes position/angles into `origin`/`angles`.
- **Side effects:** Writes to caller-provided `origin` and `angles` vectors.
- **Calls:** `G_Find`, `SpotWouldTelefrag`, `VectorSubtract`, `VectorLength`, `VectorCopy`, `G_Error`
- **Notes:** Maintains an insertion-sorted list of up to 64 spots; `rnd` is drawn from `random() * (numSpots/2)`, biasing toward the furthest half.

### SelectSpawnPoint
- **Signature:** `gentity_t *SelectSpawnPoint( vec3_t avoidPoint, vec3_t origin, vec3_t angles )`
- **Purpose:** Public entry point for non-CTF, non-spectator spawn selection; delegates entirely to `SelectRandomFurthestSpawnPoint`.
- **Notes:** Dead code block using `SelectNearestDeathmatchSpawnPoint` + retry logic exists but is commented out.

### SelectInitialSpawnPoint
- **Signature:** `gentity_t *SelectInitialSpawnPoint( vec3_t origin, vec3_t angles )`
- **Purpose:** Finds a spawn point with `spawnflags & 1` (marked "initial") for the first player entry. Falls back to `SelectSpawnPoint` if none found or telefrag risk.
- **Calls:** `G_Find`, `SpotWouldTelefrag`, `SelectSpawnPoint`, `VectorCopy`

### InitBodyQue
- **Signature:** `void InitBodyQue( void )`
- **Purpose:** Pre-allocates `BODY_QUEUE_SIZE` (8) persistent corpse entities, stored in `level.bodyQue`.
- **Side effects:** Modifies `level.bodyQueIndex`, `level.bodyQue[]`; spawns entities via `G_Spawn`.

### CopyToBodyQue
- **Signature:** `void CopyToBodyQue( gentity_t *ent )`
- **Purpose:** On respawn, copies the dying player's entity state into a pooled body entity, sets physics, animation freeze, and schedules `BodySink`.
- **Inputs:** `ent` — the live player entity about to respawn.
- **Side effects:** Modifies `level.bodyQueIndex`; links/unlinks entities; sets `body->think = BodySink` in 5 s.
- **Calls:** `trap_UnlinkEntity`, `trap_PointContents`, `trap_LinkEntity`
- **Notes:** MISSIONPACK path additionally preserves the kamikaze timer activator.

### ClientConnect
- **Signature:** `char *ClientConnect( int clientNum, qboolean firstTime, qboolean isBot )`
- **Purpose:** Called when a client begins connecting. Validates IP ban, checks password, initializes session data, connects bots, and broadcasts join message.
- **Outputs/Return:** `NULL` on success; error string to deny connection.
- **Side effects:** Writes to `level.clients[clientNum]`; calls `G_InitSessionData`/`G_ReadSessionData`; calls `CalculateRanks`.
- **Calls:** `G_FilterPacket`, `G_InitSessionData`, `G_ReadSessionData`, `G_BotConnect`, `ClientUserinfoChanged`, `BroadcastTeamChange`, `CalculateRanks`

### ClientBegin
- **Signature:** `void ClientBegin( int clientNum )`
- **Purpose:** Called once the client finishes connecting and is ready to enter the world. Calls `ClientSpawn` and fires teleport-in event.
- **Side effects:** Sets `client->pers.connected = CON_CONNECTED`; calls `CalculateRanks`.
- **Calls:** `G_InitGentity`, `ClientSpawn`, `G_TempEntity`, `CalculateRanks`

### ClientSpawn
- **Signature:** `void ClientSpawn( gentity_t *ent )`
- **Purpose:** Core spawn/respawn initializer. Selects spawn point, clears `gclient_t` while preserving persistent data, sets weapons/health/physics, runs an initial `ClientThink` frame to settle the player.
- **Side effects:** Resets most of `gclient_t`; sets entity fields; calls `trap_LinkEntity`; fires spawn point targets via `G_UseTargets`.
- **Calls:** `SelectSpectatorSpawnPoint`, `SelectCTFSpawnPoint`, `SelectInitialSpawnPoint`, `SelectSpawnPoint`, `G_KillBox`, `SetClientViewAngle`, `ClientThink`, `ClientEndFrame`, `BG_PlayerStateToEntityState`, `G_UseTargets`
- **Notes:** Uses `do { } while(1)` loop with `FL_NO_BOTS`/`FL_NO_HUMANS` flag checks to retry spawn selection.

### ClientUserinfoChanged
- **Signature:** `void ClientUserinfoChanged( int clientNum )`
- **Purpose:** Parses client userinfo string and updates server-side client state: name, handicap, model, team, colors. Broadcasts a configstring update.
- **Side effects:** Writes `level.clients[clientNum]` fields; calls `trap_SetConfigstring(CS_PLAYERS+clientNum, ...)`; logs via `G_LogPrintf`.
- **Calls:** `trap_GetUserinfo`, `ClientCleanName`, `PickTeam`, `trap_SetConfigstring`, `G_LogPrintf`

### ClientDisconnect
- **Signature:** `void ClientDisconnect( int clientNum )`
- **Purpose:** Tears down a client on drop: stops followers, spawns teleport-out effect, tosses items, clears entity, recalculates ranks, shuts down bot AI if applicable.
- **Side effects:** Unlinks entity; clears `inuse`; calls `CalculateRanks`, `BotAIShutdownClient`, `TossClientItems`.
- **Calls:** `G_RemoveQueuedBotBegin`, `StopFollowing`, `G_TempEntity`, `TossClientItems`, `CalculateRanks`, `BotAIShutdownClient`

## Control Flow Notes

- **Init:** `InitBodyQue` is called once at map load (from `g_main.c`).
- **Connect flow:** `ClientConnect` → `ClientBegin` → `ClientSpawn` → frame loop via `ClientThink`/`ClientEndFrame`.
- **Respawn:** `respawn` → `CopyToBodyQue` + `ClientSpawn`.
- **Disconnect:** `ClientDisconnect` is called by `trap_DropClient` in the server system, not directly by game logic.
- This file does not contain per-frame logic; all per-frame work is in `g_active.c`.

## External Dependencies

- **Includes:** `g_local.h` (which pulls `q_shared.h`, `bg_public.h`, `g_public.h`)
- **Defined elsewhere:**
  - `ClientThink`, `ClientEndFrame` — `g_active.c`
  - `SelectCTFSpawnPoint` — `g_team.c`
  - `BG_PlayerStateToEntityState` — `bg_misc.c`
  - `FindIntermissionPoint`, `MoveClientToIntermission` — `g_main.c` / `p_hud.c`
  - `TossClientItems`, `body_die` — `g_combat.c`
  - `G_BotConnect`, `BotAIShutdownClient` — `g_bot.c` / `ai_main.c`
  - `CalculateRanks`, `BroadcastTeamChange` — `g_main.c` / `g_cmds.c`
  - All `trap_*` functions — server syscall stubs in `g_syscalls.c`
