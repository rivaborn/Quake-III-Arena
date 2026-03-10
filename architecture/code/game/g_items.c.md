# code/game/g_items.c

## File Purpose
Implements the server-side item system for Quake III Arena, handling pickup logic, item spawning, dropping, respawning, and per-frame physics simulation for all in-game collectibles (weapons, ammo, health, armor, powerups, holdables, and team items).

## Core Responsibilities
- Execute type-specific pickup logic and award appropriate effects to the picking client
- Manage item respawn timers and team-based item selection on respawn
- Spawn world items at map load, dropping them to floor via trace
- Launch and drop items dynamically (e.g., on player death)
- Simulate per-frame physics for airborne items (gravity, bounce, NODROP removal)
- Maintain the item registration/precache bitfield written to config strings
- Validate required team-game entities (flags, obelisks) at map start

## Key Types / Data Structures
None defined locally; relies on types from `g_local.h`.

| Name | Kind | Purpose |
|------|------|---------|
| `gentity_t` | struct (extern) | Game entity — item world instance |
| `gclient_t` | struct (extern) | Client state — recipient of pickup effects |
| `gitem_t` | struct (extern, `bg_public.h`) | Item definition (type, quantity, tag, classname) |
| `level_locals_t` | struct (extern) | Global level state (time, clients, etc.) |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `itemRegistered` | `qboolean[MAX_ITEMS]` | file-static (global linkage) | Tracks which items have been registered for precaching; written to CS_ITEMS config string |

## Key Functions / Methods

### Pickup_Powerup
- **Signature:** `int Pickup_Powerup( gentity_t *ent, gentity_t *other )`
- **Purpose:** Adds powerup duration to the picking client; broadcasts a "denied" anti-reward event to nearby enemy clients with line-of-sight.
- **Inputs:** `ent` — powerup entity; `other` — picking player entity.
- **Outputs/Return:** `RESPAWN_POWERUP` (120 s).
- **Side effects:** Mutates `other->client->ps.powerups[]`; flips `PLAYEREVENT_DENIEDREWARD` bit on nearby enemy `persistant[]`; calls `trap_Trace` per nearby client.
- **Calls:** `VectorSubtract`, `VectorNormalize`, `AngleVectors`, `DotProduct`, `trap_Trace`.
- **Notes:** Syncs timer to the nearest second to keep multiple powerup countdowns aligned. Skips teammates in team game modes.

### Pickup_Holdable
- **Signature:** `int Pickup_Holdable( gentity_t *ent, gentity_t *other )`
- **Purpose:** Assigns a holdable item to the client's inventory slot; sets Kamikaze entity flag if applicable.
- **Inputs/Outputs:** Returns `RESPAWN_HOLDABLE` (60 s).
- **Side effects:** Sets `STAT_HOLDABLE_ITEM`; may set `EF_KAMIKAZE` on client.

### Pickup_Weapon
- **Signature:** `int Pickup_Weapon( gentity_t *ent, gentity_t *other )`
- **Purpose:** Grants a weapon and ammo to the client; applies respawn-rule ammo reduction when client already has partial ammo.
- **Inputs/Outputs:** Returns `g_weaponTeamRespawn` or `g_weaponRespawn` cvar value.
- **Side effects:** Sets weapon bit in `STAT_WEAPONS`; calls `Add_Ammo`; sets unlimited ammo for grappling hook.
- **Notes:** Dropped items and team-mode weapons always award full quantity.

### Touch_Item
- **Signature:** `void Touch_Item( gentity_t *ent, gentity_t *other, trace_t *trace )`
- **Purpose:** Central pickup dispatcher — validates eligibility, calls type-specific `Pickup_*`, fires events, manages respawn scheduling or entity removal.
- **Inputs:** `ent` — item; `other` — touching entity; `trace` — collision trace.
- **Side effects:** Calls type-specific pickup function; fires `EV_ITEM_PICKUP` (predictable or not); fires `EV_GLOBAL_ITEM_PICKUP` for powerups/team items; calls `G_UseTargets`; sets `nextthink`/`think = RespawnItem` or marks entity for free.
- **Calls:** `BG_CanItemBeGrabbed`, `G_LogPrintf`, `Pickup_*`, `G_AddPredictableEvent`, `G_AddEvent`, `G_TempEntity`, `G_UseTargets`, `trap_LinkEntity`.

### RespawnItem
- **Signature:** `void RespawnItem( gentity_t *ent )`
- **Purpose:** Makes an invisible item visible and collidable again; selects randomly among teamed item variants; plays respawn sounds.
- **Side effects:** Modifies `r.contents`, `s.eFlags`, `r.svFlags`; calls `trap_LinkEntity`; spawns temp sound entities; fires `EV_ITEM_RESPAWN`.

### LaunchItem
- **Signature:** `gentity_t *LaunchItem( gitem_t *item, vec3_t origin, vec3_t velocity )`
- **Purpose:** Creates a new physics-enabled dropped item entity with gravity trajectory.
- **Outputs/Return:** Pointer to the new `gentity_t`.
- **Side effects:** Allocates entity via `G_Spawn`; sets `FL_DROPPED_ITEM`; schedules `G_FreeEntity` or `Team_DroppedFlagThink` after 30 s; calls `trap_LinkEntity`.

### FinishSpawningItem
- **Signature:** `void FinishSpawningItem( gentity_t *ent )`
- **Purpose:** Finalises item placement — traces down to floor, sets collision bounds, defers powerup visibility. Called two frames after `G_SpawnItem`.
- **Side effects:** May call `G_FreeEntity` if startsolid; sets `touch = Touch_Item`, `use = Use_Item`; schedules `RespawnItem` for powerups; calls `trap_LinkEntity`.

### G_RunItem
- **Signature:** `void G_RunItem( gentity_t *ent )`
- **Purpose:** Per-frame update for airborne items — evaluates gravity trajectory, traces movement, bounces off surfaces, removes items in NODROP volumes.
- **Side effects:** Calls `trap_Trace`, `trap_LinkEntity`, `trap_PointContents`, `G_RunThink`, `G_BounceItem`, `G_FreeEntity`, `Team_FreeEntity`.
- **Calls:** `BG_EvaluateTrajectory`, `G_RunThink`, `G_BounceItem`.

### G_SpawnItem
- **Signature:** `void G_SpawnItem( gentity_t *ent, gitem_t *item )`
- **Purpose:** Initialises a map-placed item entity and defers floor-drop to `FinishSpawningItem` after 2 frames.
- **Side effects:** Calls `RegisterItem`; returns early without linking if item is disabled via cvar. Pre-caches powerup respawn sound.

### SaveRegisteredItems / ClearRegisteredItems / RegisterItem
- **Notes:** Collectively manage the `itemRegistered[]` bitfield. `SaveRegisteredItems` serialises it to config string `CS_ITEMS` via `trap_SetConfigstring` so clients know what to precache.

## Control Flow Notes
- **Init:** `G_SpawnItem` is called during map entity parsing; `FinishSpawningItem` runs on frame 3 via `nextthink`.
- **Frame:** `G_RunItem` is called each server frame for any item with non-stationary trajectory (dropped/launched items).
- **Event-driven:** `Touch_Item` fires on collision trigger contact; `RespawnItem` fires via scheduled `think` callback.
- **Shutdown/level-change:** Not handled here; item entities are freed via standard entity lifecycle.

## External Dependencies
- `g_local.h` → `q_shared.h`, `bg_public.h`, `g_public.h`
- **Defined elsewhere:**
  - `bg_itemlist`, `bg_numItems` — item table (`bg_misc.c`)
  - `BG_CanItemBeGrabbed`, `BG_FindItem`, `BG_FindItemForWeapon`, `BG_EvaluateTrajectory`, `BG_EvaluateTrajectoryDelta` — shared game library
  - `Pickup_Team`, `Team_DroppedFlagThink`, `Team_CheckDroppedItem`, `Team_FreeEntity`, `Team_InitGame` — `g_team.c`
  - `G_Spawn`, `G_FreeEntity`, `G_TempEntity`, `G_UseTargets`, `G_SetOrigin`, `G_AddEvent`, `G_AddPredictableEvent`, `G_SoundIndex`, `G_RunThink` — `g_utils.c` / `g_main.c`
  - `trap_Trace`, `trap_LinkEntity`, `trap_PointContents`, `trap_SetConfigstring`, `trap_Cvar_VariableIntegerValue`, `trap_GetUserinfo` — engine syscall stubs
  - `g_weaponRespawn`, `g_weaponTeamRespawn`, `g_gametype` — cvars declared in `g_main.c`
  - `level` — global `level_locals_t` from `g_main.c`
