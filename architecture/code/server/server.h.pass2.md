# code/server/server.h — Enhanced Analysis

## Architectural Role

This header defines the **server subsystem's contract with the engine core** (`qcommon`) and between server modules themselves. It bridges two domains: the **game VM execution layer** (authoritative game logic) and the **network replication layer** (client snapshots, reliable commands). The server is the authoritative single point of truth—all game state flows *from* the game VM, is shaped by network bandwidth constraints, and is distributed *to* connected clients. This file encodes the data structures that make that transformation efficient.

## Key Cross-References

### Incoming (who depends on this file)
- **qcommon/common.c**: Calls `SV_Frame()` once per engine tick; drives the server's tick loop
- **qcommon/vm.c**: Manages `gvm` lifecycle; calls `GAME_INIT`, `GAME_RUN_FRAME`, `GAME_SHUTDOWN` via the syscall trap table
- **client/cl_main.c**: In listen-server mode, directly calls `SV_SpawnServer()`, `SV_Frame()`, `SV_Shutdown()`; accesses `sv.svEntities`, `sv.gentities` for local client prediction
- **renderer/tr_main.c**: May read `sv.models[]` for model binding; cross-boundary texture/shader lookups
- **game/g_syscalls.c**: Game VM invokes server functions via trap table (collision, entity linking, configstrings, bot syscalls)
- **All cgame/server message handlers**: Parse `clientSnapshot_t` data sent via snapshots

### Outgoing (what this file depends on)
- **qcommon/qcommon.h**: `msg_t`, `netchan_t`, `vm_t`, cvar system, collision models (`cmodel_s`)
- **game/q_shared.h**: All shared types (`entityState_t`, `playerState_t`, `trace_t`, `usercmd_t`, configstring indices)
- **game/g_public.h**: `sharedEntity_t`, `SVF_*` flags, game VM import/export enums
- **game/bg_public.h**: Game constants, movement/item/weapon definitions
- **Implicitly via implementation**: `sv_world.c` (entity linking, area queries, traces), `sv_snapshot.c` (PVS/delta compression), botlib (via game VM syscalls)

## Design Patterns & Rationale

**Dual Time-Domain Architecture:**
- `server_t.time` and `serverStatic_t.time` decouple per-frame simulation time from persistent state. This allows `map_restart` to reset `server_t` while keeping `svs.time` monotonically increasing (no time rewinds visible to clients).

**Ring-Buffer Snapshot History:**
- `client_t.frames[PACKET_BACKUP]` and `client_t.reliableCommands[]` are fixed circular buffers. This avoids dynamic allocation on the hot path and enables O(1) delta-compression lookups: given an ack'd frame number, directly index the old state without search.

**Cluster-Based Visibility:**
- `svEntity_t` caches `clusternums[]` and `numClusters` as a packed array, avoiding per-query tree traversals. The `MAX_ENT_CLUSTERS` cap trades memory for speed: large entities that span many clusters fall back to `headnode` traversal.

**Challenge Token Anti-Spoofing:**
- `challenge_t` with time-based expiry (`firstTime`, timeout check in connect handler) prevents blind UDP reflection attacks. Attacker must receive challenge before crafting connect; legitimate client flow is: `getChallenge` → parse token → `connect <token>`.

**State Machine Strictness:**
- Client progression `CS_FREE → CS_CONNECTED → CS_PRIMED → CS_ACTIVE → CS_ZOMBIE` is unidirectional and guarded. No shortcuts—e.g., can't go directly `CS_CONNECTED` → `CS_ACTIVE` without gamestate transmission in `CS_PRIMED`.

**Separate Reliable & Unreliable Channels:**
- Configstrings and server commands flow over `netchan.reliable` (ordered, retransmitted); snapshots over unreliable (best-effort, compressed). Rationale: configstrings are small, must arrive, rarely change; snapshots are frequent, large, and transient (loss ≈ visual stutter, not fatal).

## Data Flow Through This File

```
INBOUND (per client packet):
  UDP pkt → SV_ExecuteClientMessage(client, msg)
    ├─ Parse usercmd_t (delta-compressed against lastUsercmd)
    ├─ Call SV_ClientThink(client, cmd) → game VM
    └─ Parse reliable client commands (chat, tcmd, etc)
       └─ Call SV_ExecuteClientCommand()

FRAME SIMULATION:
  SV_Frame() [called from Com_Frame, ~60Hz or sv_fps cvar]
    ├─ Advance sv.time by frame delta
    ├─ Call GAME_RUN_FRAME via gvm
    │   └─ Game writes entity states, computes damage, runs AI
    ├─ SV_BotFrame(svs.time) → botlib per-bot ticks
    └─ SV_SendClientMessages()

OUTBOUND (per-client snapshot):
  SV_SendClientMessages()
    ├─ For each client in CS_ACTIVE:
    │   ├─ SV_SendClientSnapshot(client)
    │   │   ├─ Compute PVS from player origin (CM_ClusterPVS)
    │   │   ├─ Iterate sv.svEntities[], test clusternums[] vs PVS
    │   │   ├─ Build entity list window (delta-encode against frames[ackframe])
    │   │   └─ Delta-encode playerState_t
    │   └─ SV_UpdateServerCommandsToClient (configstrings, svc_* commands)
    └─ Netchan transmission (fragmentation, rate throttling)
```

## Learning Notes

**Snapshot Replication Model** (vs. continuous streaming):
- Modern engines often send deltas per-entity on change. Q3 builds full snapshots per client per frame, PVS-culled. Trade-off: simpler consistency guarantees, but higher bandwidth on volatile maps. Snapshot frequency is configurable (`snapshotMsec` cvar).

**Configstring Versatility:**
- `configstrings` array is the primary out-of-band communication channel (map name, scores, team status, shader references). No separate "state sync" messages—the config array *is* state sync. This is idiomatic to Q3-era engines.

**Cluster-Based PVS, Not BSP-Leaf-Based:**
- Unlike some engines that cull per-BSP-leaf, Q3 uses coarser clusters (aggregates of leaves). Faster PVS queries, but slightly higher false-positive rate (more entities sent than strictly necessary).

**Entity Baseline Delta:**
- `svEntity_t.baseline` stores the first-sighting state. All future snapshots delta against this, not the previous snapshot—enabling large packet loss without propagating errors backward. (The client recomputes full state by accumulating deltas from baseline + recent snapshots.)

**Bot Integration via Syscalls, Not Linking:**
- Botlib is never statically linked; the game VM imports botlib functions via the `trap_BotLib*` syscall range (opcodes 200–599). This keeps botlib decoupled and allows it to be compiled independently (e.g., `code/bspc` uses the same botlib source offline).

## Potential Issues

- **Fixed Array Limits**: `MAX_GENTITIES`, `MAX_CONFIGSTRINGS`, `PACKET_BACKUP` are all hardcoded. A mod changing max clients or entity count requires recompilation. Modern engines use dynamic pools.
- **Server Shutdown Coupling**: `SV_DropClient()` triggers game VM callbacks; if game VM crashes, client cleanup may leak. No guard against cascading disconnects.
- **Challenge Table Exhaustion**: `MAX_CHALLENGES` (1024) is small; a flood of invalid connects could exhaust it in seconds on a loaded public server. No priority queue for legitimate challenges.
- **Cluster Overflow Fallback**: If `numClusters > MAX_ENT_CLUSTERS`, the code falls back to `headnode` traversal, silently degrading performance. Callers may not notice.

---

**Key Takeaway:** `server.h` is a **boundary-spanning integration layer**. It doesn't implement logic; it *mediates* between the deterministic game VM (which sees unlimited CPU) and the bandwidth-constrained network (which sees ~100–1000 entity updates per snapshot). Understanding this file means understanding how Quake III partitions the simulation: authoritative state in the VM, visibility culling and rate throttling at the network boundary, and client-side prediction as best-effort local smoothing.
