# code/game/g_items.c — Enhanced Analysis

## Architectural Role

`g_items.c` implements the item system subsystem within the **server-side game logic VM** (`code/game`). It bridges the **Entity System** (entity lifecycle, spatial partitioning, physics simulation) with **game rules** (scoring, client state mutation, team objectives). All item spawning, pickup validation, respawning, and physics simulation for airborne items flow through this file, making it the central coordinator between the **collision/physics engine**, **client communication** (events/sounds), and **game state** (inventory, powerup timers, team rules).

## Key Cross-References

### Incoming (who depends on this file)

- **Entity System:** `G_RunFrame` (in `g_main.c`) calls `G_RunItem()` once per frame for each airborne item entity.
- **Collision Engine (qcommon):** `trap_Trace` callbacks trigger `Touch_Item()` when a player collides with an item's trigger volume.
- **Entity Spawn Dispatcher:** `G_SpawnItem()` is called during BSP entity string parsing from `G_Spawn()` (via entity class table dispatch).
- **Map Initialization:** `G_InitGame()` (in `g_main.c`) calls `SaveRegisteredItems()` to broadcast the `CS_ITEMS` configstring after all map items are loaded.
- **Team System:** `g_team.c` calls `Team_CheckDroppedItem()` (defined in this file as a wrapper) and expects `Pickup_Team()` to handle flag pickups.

### Outgoing (what this file depends on)

- **Entity Lifecycle:** Calls `G_Spawn()`, `G_FreeEntity()`, `G_SetOrigin()` (in `g_utils.c`).
- **Physics & Collision:** Uses `trap_Trace()`, `trap_LinkEntity()`, `trap_PointContents()` for trajectory evaluation and spatial registration.
- **Event Broadcasting:** Calls `G_TempEntity()`, `G_AddEvent()`, `G_AddPredictableEvent()`, `G_SoundIndex()` to emit pickup/respawn events to clients.
- **Game Rules Trigger:** Calls `G_UseTargets()` after item pickup to activate targetname-linked entities.
- **Shared Game Library (bg_*):** Calls `BG_CanItemBeGrabbed()`, `BG_EvaluateTrajectory()`, `BG_EvaluateTrajectoryDelta()`, reads `bg_itemlist[]`, `bg_numItems`.
- **Global Level State:** Reads `level.time`, `level.maxclients`, `level.clients[]` for respawn timing and event broadcasting logic.
- **CVars:** Reads `g_gametype`, `g_weaponRespawn`, `g_weaponTeamRespawn` for respawn rule decisions.
- **Team Module (g_team.c):** Calls `Pickup_Team()`, `Team_DroppedFlagThink()`, `Team_CheckDroppedItem()`, `Team_FreeEntity()` for team-specific item logic.

## Design Patterns & Rationale

### 1. **Type-Dispatched Pickup Handler Pattern**
`Touch_Item()` uses a `switch(ent->item->giType)` to invoke type-specific `Pickup_*()` handlers (Weapon, Ammo, Armor, Health, Powerup, Holdable, Team). This is idiomatic for 1990s game engines predating virtual dispatch or trait-based designs. Each pickup type returns a respawn duration; this indirection allows specialized logic (e.g., weapon ammo reduction) without bloating a monolithic handler.

### 2. **Think-Based Scheduling**
Respawning uses Quake's classic `nextthink`/`think` callback pattern rather than explicit timers. `RespawnItem()` is scheduled as a think function to run at a specific frame. This design choice:
- Avoids maintaining a separate timer array
- Leverages the existing entity think-update loop
- Keeps state localized to the entity
- Is inherited from Quake 1/2 and baked into Q3A's architecture

### 3. **Teamed Item Selection**
`RespawnItem()` randomly selects from teamed items by traversing `teammaster → teamchain` and using modulo-based selection. This pattern ensures map symmetry in competitive play (e.g., one of two power-armor spawns respawns, chosen randomly).

### 4. **Trajectory-Based Physics**
Dropped items use `BG_EvaluateTrajectory()` to compute position each frame, rather than accumulating per-frame forces. This design:
- Reduces per-frame cost (single math evaluation vs. integration)
- Ensures deterministic physics across client prediction and server simulation
- Is inherited from Quake's trajectory structure design

### 5. **Client-Side Prediction Support**
Item pickups are split: some events are "predictable" (sent via `G_AddPredictableEvent`, e.g., armor) and some are not (powerups always server-authoritative via `G_AddEvent`). This allows clients to speculatively show pickup feedback without cheating on game-critical items.

### 6. **Smart Weapon Ammo Respawn**
`Pickup_Weapon()` implements a respawn-rule reduction: if the player already has ammo, only 1 shot is awarded on respawn (instead of full quantity). This competitive balance feature prevents runaway weapons for players holding positions. Dropped items and team-mode weapons bypass this (always full ammo).

## Data Flow Through This File

**Phase 1: Map Load & Precaching**
```
1. G_SpawnItem() called during entity string parsing
   → RegisterItem() marks item as registered
   → Defers FinishSpawningItem() via nextthink (frame+2)
   
2. Frame N+2: FinishSpawningItem() runs
   → Traces down to find floor
   → Sets touch handler & use handler
   → For powerups: schedules RespawnItem via nextthink
   → trap_LinkEntity() registers with spatial partition
   
3. SaveRegisteredItems() called after all entities loaded
   → Encodes itemRegistered[] bitfield
   → Writes to CS_ITEMS configstring for client precache
```

**Phase 2: Dynamic Item Drop (on player death, weapon drop, etc.)**
```
LaunchItem() called (e.g., from G_Damage → drop weapon)
   → G_Spawn() creates new entity
   → Sets FL_DROPPED_ITEM flag
   → Schedules G_FreeEntity or Team_DroppedFlagThink after 30s
   → trap_LinkEntity()
   
Each frame:
   → G_RunItem() updates dropped item
   → BG_EvaluateTrajectory() computes new position (gravity applied)
   → trap_Trace() moves to new position, detects collisions
   → G_BounceItem() bounces off surfaces if needed
   → trap_PointContents() checks for NODROP volumes (lava, etc.)
   → If in NODROP → G_FreeEntity() or Team_FreeEntity()
   → Eventually comes to rest or is freed
```

**Phase 3: Pickup**
```
Player touches item → Touch_Item() called (collision callback)
   → BG_CanItemBeGrabbed() validates (team rules, CTF flags, etc.)
   → G_LogPrintf() logs pickup
   → Type-specific Pickup_*() handler executes:
      - Modifies client→ps.stats (STAT_WEAPONS, STAT_HEALTH, etc.)
      - Returns respawn duration or 0
   
   → If respawn > 0:
      - Entity made invisible/untouchable (EF_NODRAW, ~CONTENTS_TRIGGER)
      - nextthink scheduled for RespawnItem()
   
   → Fire events:
      - G_AddPredictableEvent (for items not cheat-sensitive)
      - G_AddEvent (for powerups)
      - EV_GLOBAL_ITEM_PICKUP temp entity (broadcast to far clients)
   
   → G_UseTargets() triggered (e.g., "drop weapon pickup triggers fog")
```

**Phase 4: Respawn**
```
nextthink fires → RespawnItem() runs
   → If teamed: random selection from teamchain
   → Make visible & collidable (clear EF_NODRAW, ~SVF_NOCLIENT)
   → Fire respawn sound events (EV_GENERAL_SOUND or EV_GLOBAL_SOUND)
   → G_AddEvent(EV_ITEM_RESPAWN)
   → trap_LinkEntity() to re-register
```

## Learning Notes

### Quake-Era Idioms in Modern Context
This file demonstrates several design choices that were optimal in the late 1990s but differ from modern engines:

1. **Synchronous Physics Callbacks:** `Touch_Item()` is a synchronous collision callback, not an event queue. Modern engines often decouple collision detection from response via event systems.

2. **Trajectory Structs vs. Continuous Simulation:** Rather than per-frame force integration, Q3A pre-computes trajectories. Modern engines (Unity, Unreal) use physics engines with continuous solvers.

3. **VM/Native Boundary:** The entire system works identically as QVM bytecode or native DLL via `trap_*` syscalls. Modern engines embed physics and scripting together.

4. **Configuration Strings for Asset Broadcasting:** Clients learn what to precache via CS_ITEMS configstring. Modern engines use asset manifests or on-demand streaming.

5. **Think Callback Pattern:** `nextthink` scheduling is a lightweight alternative to coroutines or task schedulers. Modern engines use async/await or component-based tick systems.

### Integration with Larger Systems
- **Entity System:** g_items.c is a thin layer atop the entity lifecycle. It spawns, updates, and frees entities like any other system.
- **Prediction:** The split between predictable and authoritative pickups shows Q3A's prediction model: clients speculatively run non-critical game code, server corrects authoritative state.
- **Team Rules:** Flag/obelisk pickups are tightly coupled to `g_team.c`. Modifying team rules requires changes here.
- **Bot AI:** Bots use the same item spawn/respawn system as players; botlib navigates to item spawn points from AAS data.

## Potential Issues

1. **MISSIONPACK Divergence:** Multiple `#ifdef MISSIONPACK` blocks (e.g., `Pickup_PersistantPowerup`, armor cap logic in `Pickup_Armor`) create maintenance burden and risk of drift between code paths.

2. **Powerup Event Broadcasting Overhead:** `Pickup_Powerup()` traces line-of-sight to ALL nearby clients to deliver anti-reward feedback. With many players, this O(N) loop becomes expensive.

3. **Hardcoded Timing Constants:** Respawn durations (RESPAWN_ARMOR=25, RESPAWN_POWERUP=120) are compile-time constants, not cvars. This prevents fine-grained server configuration.

4. **Team Item Coupling:** Flags and obelisks are implemented as special-case items with callbacks in `Team_DroppedFlagThink`. If new team mechanics are added, this file becomes a bottleneck.

5. **No Item Damage/Destruction:** Dropped items don't take damage or get destroyed by explosions; they only vanish via respawn timer or NODROP volumes. This limits dynamic item destruction scenarios.

6. **Global itemRegistered Array:** File-static `itemRegistered[MAX_ITEMS]` array is simple but not scalable if the item table grows beyond ~256 entries (one byte per item).

---

**Sources Referenced:**
- First-pass analysis (function-level breakdown)
- Architecture context (subsystem integration points)
- Cross-reference excerpts (incoming/outgoing dependencies)
