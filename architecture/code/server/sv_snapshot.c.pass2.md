# code/server/sv_snapshot.c — Enhanced Analysis

## Architectural Role

`sv_snapshot.c` is the primary bandwidth-efficiency layer of the authoritative server. It sits at the boundary between the game VM's in-memory world state (`sv.svEntities[]`, `playerState_t`) and the UDP network channel, performing the spatial culling and delta-encoding that make Q3A's network model practical over late-1990s internet connections. It serves as the one-way valve through which the server's ground-truth state becomes client-observable state consumed by `cgame` via `cl_parse.c → cg_snapshot.c`. Without this file the client receives no game world updates; with it the server sends only what each client can see, encoded as a diff against what they already know.

## Key Cross-References

### Incoming (who depends on this file)

- **`sv_main.c` / `SV_Frame`** → calls `SV_SendClientMessages` once per server frame (~20 Hz tick). This is the sole trigger for the entire snapshot pipeline.
- **`sv_client.c` / connection priming** → calls `SV_UpdateServerCommandsToClient` directly when sending the initial `CS_PRIMED` configstring flush, before the client enters active state.
- **`sv_bot.c`** → calls `SV_BuildClientSnapshot` for bot clients to populate their frame state; bots then read `svs.snapshotEntities` and `client->frames` directly rather than going through the network encoder. This makes `SV_BuildClientSnapshot` the shared contract between the real-net path and the bot shortcut path.
- **`code/game` (via `trap_GetSnapshot`)** → the game VM can query snapshot data for each client through the engine syscall boundary in `sv_game.c`; the data source is the frame ring built here.

### Outgoing (what this file depends on)

- **`qcommon/msg.c`** — `MSG_WriteDeltaEntity`, `MSG_WriteDeltaPlayerstate`, `MSG_WriteByte/Long/Bits/Data/String/Init/Clear` — all serialization is delegated here. The delta-compression algorithms themselves live in `msg.c`, not in this file.
- **`qcommon/cm_*.c`** — `CM_PointLeafnum`, `CM_LeafArea`, `CM_LeafCluster`, `CM_ClusterPVS`, `CM_AreasConnected`, `CM_WriteAreaBits` — every PVS and area-portal query routes through the collision model. The server shares the same PVS data used by the renderer's `tr_world.c` culling pass.
- **`sv_game.c`** — `SV_GentityNum`, `SV_GameClientNum`, `SV_SvEntityForGentity` — translates between game VM entity indices and server-side `svEntity_t` / `sharedEntity_t` pointers.
- **`sv_net_chan.c`** — `SV_Netchan_Transmit`, `SV_Netchan_TransmitNextFragment` — the finalized `msg_t` buffer is handed off here for reliable-sequenced UDP delivery.
- **`sv_client.c`** — `SV_WriteDownloadToClient` — download chunks are appended after the snapshot payload in `SV_SendClientSnapshot`.
- **Globals read/written:** `sv.snapshotCounter`, `sv.svEntities[]`, `sv.num_entities`; `svs.time`, `svs.snapshotEntities[]`, `svs.nextSnapshotEntities`, `svs.numSnapshotEntities`, `svs.clients[]`; cvars `sv_padPackets`, `sv_maxRate`, `sv_lanForceRate`, `sv_maxclients`.

## Design Patterns & Rationale

**Two-pointer sorted merge for delta encoding:** `SV_EmitPacketEntities` uses a classic merge-of-two-sorted-lists pattern (sentinel `9999` at exhaustion). This is necessary because `svs.snapshotEntities` is a circular ring whose entries for a given frame are ordered by entity number after the `qsort` in `SV_BuildClientSnapshot`. The merge allows `MSG_WriteDeltaEntity` to produce minimal diffs: unchanged entities emit zero bytes (the `force=qfalse` path in `msg.c`), added entities delta from their static baseline in `sv.svEntities[n].baseline`, and removed entities are tombstoned.

**Snapshot counter for O(1) dedup:** `sv.snapshotCounter` (bumped once per `SV_BuildClientSnapshot` call) marks each `svEntity_t` on first addition. Re-encountering a marked entity (from a portal recursion or a broadcast flag) is a single integer compare — no visited-set allocation. This is a classic "timestamp as set membership" trick for graph traversal.

**Circular frame ring + circular entity buffer:** `client->frames[PACKET_BACKUP]` and `svs.snapshotEntities[svs.numSnapshotEntities]` are dual rings. The frame ring allows delta-compression against any of the last ~32 frames; the entity buffer amortizes per-snapshot allocation. The validity check `oldframe->first_entity <= svs.nextSnapshotEntities - svs.numSnapshotEntities` is the guard against using a frame whose entities have been overwritten.

**Rate throttling as a first-class concern:** `SV_SendMessageToClient` distinguishes LAN/loopback (always send immediately), non-active clients (slow poll at 1 Hz unless downloading), and active clients (governed by `sv_maxRate` and `snapshotMsec`). This graduated policy reflects the bandwidth reality of 1999 modem/DSL players.

**Tradeoff — portal visibility depth:** `SV_AddEntitiesVisibleFromPoint` is mutually recursive with no depth guard. In practice the level designer must avoid infinite portal chains, but the engine offers no hard backstop.

## Data Flow Through This File

```
SV_Frame (sv_main.c)
  └─ SV_SendClientMessages
       └─ [per active client, if nextSnapshotTime elapsed]
            SV_SendClientSnapshot
              ├─ SV_BuildClientSnapshot          ← mutates sv/svs state
              │    ├─ SV_GameClientNum → playerState_t copy → frame->ps
              │    ├─ SV_AddEntitiesVisibleFromPoint
              │    │    ├─ CM_* → PVS bits + area bits → frame->areabits
              │    │    └─ SV_AddEntToSnapshot → entityNumbers[]
              │    ├─ qsort(entityNumbers)
              │    ├─ XOR-invert areabits (visible→mask)
              │    └─ copy entityState_t[] → svs.snapshotEntities ring
              │
              ├─ MSG_Init (msg_t scratch buffer, ~16KB)
              ├─ SV_UpdateServerCommandsToClient → svc_serverCommand writes
              ├─ SV_WriteSnapshotToClient
              │    ├─ delta-select oldframe (or NULL for full send)
              │    ├─ MSG_WriteDeltaPlayerstate(oldframe->ps, frame->ps)
              │    └─ SV_EmitPacketEntities(oldframe, frame, msg)
              │         └─ MSG_WriteDeltaEntity × N
              ├─ SV_WriteDownloadToClient (if downloading)
              └─ SV_SendMessageToClient → SV_Netchan_Transmit
```

Key state transitions: `svs.nextSnapshotEntities` advances monotonically (never wraps in practice — `Com_Error(ERR_FATAL)` if it would); `client->nextSnapshotTime` is set forward by rate logic; `client->rateDelayed` flag is reflected in `snapFlags` so the client can adjust interpolation.

## Learning Notes

**What a developer learns here:**
- The complete Q3A network model in one file: PVS culling → entity list diff → delta bitstream → rate-throttled UDP. Each layer is independently readable.
- How `areabits` work as a bitmask of connected areas XOR-inverted for the renderer — the inversion is non-obvious and easy to get wrong when porting.
- The baseline mechanism: every entity has a zero-initialized `baseline` stored server-side; new-to-frame entities always diff against this rather than a previous frame, ensuring correctness even on the first send.

**Era-idiomatic patterns modern engines handle differently:**
- Manual sorted entity lists instead of spatial acceleration (octree/BVH) for visibility. At Q3A's entity counts (~256 max), the linear scan is competitive; modern games with thousands of entities use spatial partitioning and GPU-assisted occlusion.
- Per-client snapshots fully computed on the CPU, one per client per frame. Modern engines often use a shared "world snapshot" and apply per-client filtering as a post-pass, or offload to dedicated snapshot worker threads.
- No concept of interest management beyond PVS + area — modern competitive games layer bandwidth priority, relevancy decay, and LOD-based update rates on top of pure visibility.
- `SVF_CLIENTMASK` capped at 32 players (hardcoded `clientNum >= 32` check) — reflects Q3A's 16-player tournament design; extending this would require rethinking the bitmask type.

**Connection to broader concepts:** The snapshot + delta-compression design is a textbook implementation of the "state synchronization" networking model (as opposed to "event/command" replication). The per-frame `snapshotCounter` dedup technique generalizes to any frame-coherent graph traversal problem.

## Potential Issues

- **Silent entity drop at MAX\_SNAPSHOT\_ENTITIES (1024):** `SV_AddEntToSnapshot` discards entities beyond 1024 with no log or counter. A map with many active entities (e.g., heavy rocket fire) can silently omit entities from client view.
- **`SVF_CLIENTMASK` 32-client hard limit:** The `Com_Error(ERR_DROP)` at `clientNum >= 32` would disconnect a player in a hypothetical 64-player mod without any compile-time guard. The `singleClient` field reused as a bitmask is a latent ABI design hazard.
- **`FIXING ENT->S.NUMBER!!!` debug print:** The live fixup in `SV_AddEntitiesVisibleFromPoint` (`ent->s.number = e`) suggests the game VM can corrupt entity numbers, and the server silently patches it each snapshot build — a symptom that the VM/engine shared-memory boundary for entity state lacks enforcement.
- **Unbounded portal recursion:** No depth counter in `SV_AddEntitiesVisibleFromPoint` recursive calls. A misconfigured portal entity pair referencing each other would overflow the stack.
