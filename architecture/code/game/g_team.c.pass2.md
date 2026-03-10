# code/game/g_team.c — Enhanced Analysis

## Architectural Role

This file implements the server-authoritative team game rules layer for CTF and One-Flag-CTF modes, bridging the game logic VM with the engine's entity and networking systems. It manages the complete lifecycle of flag entities (spawn → carry → drop → return/capture), computes spatially-aware frag bonuses tied to map geography, and orchestrates team-wide state synchronization to clients via config strings and broadcast messages. In MISSIONPACK modes (Obelisk, Harvester), it additionally owns the obelisk entity lifecycle and shared objective damage tracking.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/game/g_combat.c`** → calls `Team_FragBonuses` and `Team_CheckHurtCarrier` during kill/damage events to award team bonuses and detect carrier assaults
- **`code/game/g_items.c`** → calls `Pickup_Team` when a player touches a team item; also registers `Team_DroppedFlagThink` as the think callback for dropped flag entities
- **`code/game/g_active.c`** → calls `CheckTeamStatus` every server frame to sample and broadcast per-client team location data
- **`code/game/g_spawn.c`** → indirectly spawns team objectives via entity string dispatch; likely calls registered spawn callbacks for flags and obelisks
- **`code/game/g_main.c`** (module init) → calls `Team_InitGame` at map start to reset team state and push initial flag statuses

### Outgoing (what this file depends on)

- **`code/game/g_combat.c`** → calls `AddScore` for all bonus awards; calls `CalculateRanks` after flag captures
- **`code/game/g_items.c`** → calls `G_Find` to locate flag/obelisk entities by classname; calls `RespawnItem` to reset dropped flags
- **`code/game/g_utils.c`** → calls `G_TempEntity` to spawn ephemeral team-sound event entities; calls `G_Spawn` and `G_FreeEntity` for obelisk lifecycle
- **`code/qcommon` (engine)** → calls `trap_SetConfigstring` to push flag status to all clients; calls `trap_SendServerCommand` for team messages; calls `trap_InPVS` and `trap_Trace` for spatial queries
- **Global state** → reads/writes `level` (frame time, team scores, gametype), `g_entities[]` (all entities), `g_gametype` and `g_obelisk*` cvars (MISSIONPACK)
- **Utility functions** → calls `OnSameTeam`, `OtherTeam`, `TeamName` (string lookups), `PrintMsg` (broadcast), `SelectSpawnPoint`, `SpotWouldTelefrag` for spawn logic

## Design Patterns & Rationale

**Singleton Pattern with Thin Client Sync**  
The `teamgame` global holds all mutable team state and is serialized to a compact config string (`CS_FLAGSTATUS`) sent to all clients. This minimizes network overhead: flag status changes are atomic single-string updates rather than individual entity deltas. The remap arrays (`ctfFlagStatusRemap`, `oneFlagStatusRemap`) compress 5 states into a 2-character string.

**Event-Driven Bonuses via Spatial Proximity**  
Rather than maintain explicit "defender" roles, the engine computes bonuses reactively after each frag by checking attacker/target positions relative to flag/carrier using vector subtraction and PVS visibility. This reduces bookkeeping but requires O(n) distance calculations per frag.

**Conditional Feature Compilation**  
MISSIONPACK obelisk and harvester modes are cleanly gated by `#ifdef MISSIONPACK`, allowing the base CTF code to remain unchanged. This modular design allowed id Software to release Team Arena as an expansion without forking the entire codebase.

**Callback-Based Entity Lifecycle**  
Dropped flags and obelisks register think/pain/die callbacks at spawn time, following the Quake engine's entity simulation model. No explicit polling loop—the server frame loop calls each entity's think function each tick.

## Data Flow Through This File

**Flag Lifecycle:**
1. **Spawn** → Map entity spawned via entity string → `Pickup_Team` registers the touch callback
2. **Pickup** → Player touches enemy flag → `Team_TouchEnemyFlag` sets powerup on player, updates `teamgame.redStatus`/`blueStatus`, triggers sound event
3. **Carry** → Player moves with flag, server broadcasts position via snapshot each frame (no g_team.c involvement during carry)
4. **Drop** → Player dies/drops item → `Team_DroppedFlagThink` runs each frame, `Team_CheckDroppedItem` sets `FLAG_DROPPED` status
5. **Return** → Player or timeout returns flag to base → `Team_TouchOurFlag` resets status to `FLAG_ATBASE`
6. **Capture** → Enemy flag carrier touches own base → `Team_TouchOurFlag` increments score, triggers capture sound, resets both flags via `Team_ResetFlags`

**Obelisk Lifecycle (MISSIONPACK):**
1. **Spawn** → `SpawnObelisk` allocates entity, registers pain/die callbacks
2. **Damage In** → `CheckObeliskAttack` gate-keeps friendly fire, rate-limits attack sounds
3. **Pain** → `obelisk_pain` plays sound, updates attack timestamp
4. **Death** → `obelisk_die` plays death sound, grants bonus to killer, respawns after delay
5. **Respawn** → `obelisk_reset` repositions and heals entity

**Team Overlay Broadcasting:**
- Every `TEAM_LOCATION_UPDATE_TIME` ms, `CheckTeamStatus` walks all clients, looks up their location index via `Team_GetLocation`, and sends a `TeamplayInfoMessage` with health/armor/weapon/position data to teammates only.

## Learning Notes

**What Developers Learn Here:**
- **Network Optimization**: How to compress dynamic game state into minimal footprint (single config string for CTF flag status) while maintaining responsiveness
- **Spatial Reasoning in Games**: Proximity-based bonus logic shows how distance checks and PVS visibility combine to create emergent gameplay (e.g., defending your flag base grants bonuses only within a radius)
- **Modular Expansion**: The `#ifdef MISSIONPACK` pattern demonstrates how feature branches remain maintainable—Harvester/Obelisk code is visually isolated from base CTF logic
- **Callback Dispatch Architecture**: The entity think/pain/die model is fundamental to the Quake engine; understanding g_team.c's use of these callbacks clarifies the broader entity simulation paradigm
- **Event-Driven Messaging**: Use of temp entities and config strings to communicate state changes to all clients without explicit polling or per-entity update logic

**Era-Specific Patterns:**
Modern engines would likely use: ECS for flag/obelisk components (not global state), spatial hashing for proximity queries (not per-frag O(n) checks), event buses instead of scattered `PrintMsg` calls, and data-driven game rules (not hardcoded bonus values). The singleton + callback model works well for Quake's scale but doesn't scale to 100+ networked entities.

## Potential Issues

**Bug in `Team_FragBonuses` (line ~415–417):**  
In the carrier-protect radius check, the second distance calculation reuses `v1`:
```c
VectorSubtract(targ->r.currentOrigin, carrier->r.currentOrigin, v1);
VectorSubtract(attacker->r.currentOrigin, carrier->r.currentOrigin, v1);  // should be v2
```
The attacker distance should use `v2`, not overwrite `v1`. This causes the attacker's distance check to use the target's distance value, potentially awarding bonuses incorrectly.

**High Coupling to Global State:**  
All functions read `g_gametype`, `level.time`, `level.teamScores`, `g_entities[]` directly with no abstraction. Changes to these globals are not locally visible, making refactoring difficult. A modern codebase would inject or cache these as function parameters.

**Linear Search for Carriers (O(maxclients)):**  
`Team_FragBonuses` walks all clients to find the flag carrier each frag instead of caching a pointer. For small player counts this is fine, but it's inefficient compared to direct entity lookup.
