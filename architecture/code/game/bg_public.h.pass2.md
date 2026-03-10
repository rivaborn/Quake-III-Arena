# code/game/bg_public.h — Enhanced Analysis

## Architectural Role
This file is the **VM boundary contract** between the server-side game module (`code/game/`, running in the `gvm` QVM) and the client-side cgame module (`code/cgame/`, running in the `cgame` QVM). It defines shared enums, state indices, and network-transmittable constant vocabularies that allow deterministic **client-side movement prediction** to match server authority. The shared `bg_*.c` files compile into *both* VMs with identical implementation, ensuring the `Pmove` function produces byte-for-byte identical results on server and client when given the same input, eliminating prediction drift.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/game/g_active.c` / `g_client.c`**: Runs `Pmove(pmove_t*)` authoritatively each server frame; updates `playerState_t` and fires events via `BG_AddPredictableEventToPlayerstate`.
- **`code/cgame/cg_predict.c`**: Runs `Pmove(pmove_t*)` speculatively on unacknowledged `usercmd_t`s; detects divergence from server snapshot and applies decay corrections.
- **`code/cgame/cg_event.c`**: Consumes `entity_event_t` values fired into `playerState_t->events[]` and translates to audio/visual effects (footsteps, gunfire, death screams, award sprites).
- **`code/cgame/cg_ents.c`**: Uses `entityType_t` classification and `entity_state_t->eFlags` bit masks to dispatch per-frame entity rendering.
- **`code/game/g_items.c`** and **`code/cgame/cg_snapshot.c`**: Query `bg_itemlist[]` via `BG_Find*` helpers; validate item pickup eligibility via `BG_CanItemBeGrabbed`.
- **`code/server/sv_snapshot.c`**: Uses `statIndex_t` and `persEnum_t` indices to build/delta-compress `playerState_t` snapshots for network transmission.
- **`code/client/cl_parse.c`**: Deserializes snapshots containing entity states and applies `CS_*` configstring indices for global state.
- **UI VMs** (`code/q3_ui/` and `code/ui/`): Consume `weapon_t`, `powerup_t`, `holdable_t` enums for inventory display; may query via `BG_Find*` for item names in menus.

### Outgoing (what this file depends on)
- **`q_shared.h`** (imported): Defines `playerState_t`, `entityState_t`, `usercmd_t`, `trajectory_t`, `vec3_t`, `qboolean`, and collision constants (`CONTENTS_*`, `MASK_*`); defines `CS_SERVERINFO`/`CS_SYSTEMINFO` base indices; defines `MAX_MODELS`, `MAX_SOUNDS`, `MAX_CLIENTS`, `MAX_LOCATIONS`.
- **`be_*.h` headers** (game VM only, not cgame): Botlib interfaces used in `code/game/g_bot.c` and `ai_*.c` files; included transitively via `botlib.h` in game module.
- **Shared implementation** (`code/game/bg_pmove.c`, `bg_misc.c`, `bg_lib.c`, `q_math.c`, `q_shared.c`): Compiled into both VMs; no runtime interdependency (link-time inclusion ensures determinism).

## Design Patterns & Rationale

### 1. **Callback-Driven Physics** (`pmove_t`)
The `pmove_t` struct holds function pointers (`trace`, `pointcontents`) rather than embedding a trace library. This allows identical movement code (`bg_pmove.c`) to run on server (using authoritative BSP collision) and client (using client-side collision copy), critical for prediction correctness.

### 2. **Enum-Based Constants Over Bitmasks**
Config-string indices (`CS_MODELS`, `CS_SOUNDS`, etc.) and entity/player state indices (`statIndex_t`, `persEnum_t`) are `#define` enums, not `bitfield` structs. This simplifies network serialization: indices serialize as integers, not bit-packed fields.

### 3. **Preprocessor Gating for Expansion Packs** (`#ifdef MISSIONPACK`)
The base Q3A (`GT_FFA`, `GT_CTF`, 9 weapons) is complemented by Team Arena additions (`WP_NAILGUN`, `PW_SCOUT`, `EF_TICKING`). The ifdef gate allows a single source tree to build both. However, this couples game logic variants at compile time—mixing base and MissionPack clients/servers causes desync.

### 4. **Global Item List with Accessor Pattern**
`bg_itemlist[]` is a global array defined in `bg_misc.c`, exported as `extern` here. Clients find items via `BG_FindItem()`, `BG_FindItemForWeapon()`, etc., which use linear search. The `ITEM_INDEX(ptr)` macro converts a `gitem_t*` back to its array index via pointer arithmetic—simple but only works if the caller keeps a stable array pointer.

### 5. **Event Serialization Without Duplication**
`BG_AddPredictableEventToPlayerstate()` enqueues events in a ring buffer (`playerState_t->events[MAX_PS_EVENTS]`). Both server and client call this with the same inputs, so events don't duplicate over the network—only the event ring is delta-encoded.

## Data Flow Through This File

**Server authority → Client prediction:**
1. Server runs `Pmove(&pmove_t {.ps = &ent->player_state, .cmd = ucmd, ...})` → mutates player state and fires events
2. Game serializes `playerState_t` + `entityState_t` + events into snapshot; queues for transmission
3. Server sends snapshot (compressed) + unreliable `svc_*` configstrings for global state
4. Client deserializes snapshot; updates `cg.snap` (server's view)
5. Client predicts forward: runs `Pmove(&pmove_t {.ps = &cg.predictedPlayerState, ...})` with unacknowledged `usercmd_t`s
6. If predicted state diverges from server snapshot, apply lerp decay to hide correction
7. Cgame renders interpolated entity positions and fires events (sounds, particles, score popups) triggered by `entity_event_t` values

**Config-string indices:**
Provide a shared vocabulary for critical strings (map name, voting state, flag status). Sent unreliably but with a sequence counter; a lost configstring is re-sent on next snapshot.

## Learning Notes

### Idiomatic to This Era (1999–2005)
- **Direct VM boundary.** No RPC or message-passing framework; game and cgame are two separate QVM instances with a shared import/export vtable. State crossing must be serialized explicitly (snapshots, configstrings).
- **Callbacks instead of async events.** The `pmove_t` trace callbacks are synchronous function pointers; modern engines use async pathfinding with promise/async-await.
- **Enum indices over UUIDs.** All meaningful objects (items, weapons, powerups, entity types, events) are identified by small enum indices. Network-efficient but tightly coupled—adding a new weapon requires shifting all later enum values.
- **No object-oriented entity hierarchy.** There is no `Entity` base class or virtual dispatch; `entityType_t` values are used for type discrimination, and rendering/logic dispatch happens via switch statements in `cg_ents.c` and `g_active.c`.
- **Client-side prediction determinism.** A unique feature of Q3A's design: the cgame VM can predict movement/physics *exactly* by running the same Pmove code. Modern engines typically don't expose physics to the client for anti-cheat reasons.

### Modern Engines Do Differently
- **Server-authoritative positioning with interpolation.** No client-side physics prediction; the server sends a stream of entity snapshots, and the client interpolates visually. Eliminates prediction divergence but incurs ~100ms input lag (acceptable in 2025, not in 1999).
- **Binary message formats or Protocol Buffers** instead of hand-rolled Huffman-compressed bitfields and `MSG_WriteBits()`.
- **Centralized asset registry.** Rather than a global `bg_itemlist[]` and `ITEM_INDEX()` macro tricks, a modern engine would use handle-based or GUID-based asset references.

### Connection to ECS / Scene Graphs
Q3A predates both patterns. It uses a flat entity array with a sector tree for spatial queries; no explicit scene graph or hierarchical transforms (though `BG_PlayerStateToEntityState()` effectively flattens player animation state to networked position/rotation).

## Potential Issues

1. **Config-string overflow.** `CS_MAX` is checked against `MAX_CONFIGSTRINGS` (1024 by default in `qcommon.h`). If a mod adds too many models/sounds/clients, the compile-time check fails—but only at build time, not gracefully at runtime.

2. **MISSIONPACK coupling.** Code conditionally compiled on `#ifdef MISSIONPACK` must be tested on both branches. A MissionPack binary connecting to a base-Q3A server (or vice versa) will desync if the weapon/powerup enums diverge.

3. **`bg_itemlist[]` pointer arithmetic.** The `ITEM_INDEX()` macro assumes the `gitem_t*` is within `bg_itemlist`'s allocated block. A stale or corrupt pointer will produce out-of-bounds indices. No bounds checking.

4. **Silent event loss on buffer full.** `playerState_t->events[]` is fixed size (`MAX_PS_EVENTS = 4` in `q_shared.h`). If the server fires more than 4 events in a frame, the ring buffer overwrites—no warning.

5. **Animation enum is not forward-compatible.** The `animNumber_t` enum directly indexes frame data in the MD4/MD3 skeletal model loader. Adding an animation in the middle of the enum requires rebuilding all player models and shaders.

---

**Integration note:** This header's power lies in its role as the **contract enforcement layer**. Changes here ripple through both VMs: a new entity flag requires updates to render dispatch, prediction logic, and network delta compression. The design reflects Q3A's era of monolithic game codebases; modern engines would split this into versioned network protocol definitions and per-subsystem API contracts.
