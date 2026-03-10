# code/server/sv_client.c — Enhanced Analysis

## Architectural Role

`sv_client.c` is the **connection gateway and in-game protocol interpreter** of the Server subsystem. It sits precisely at the junction of four subsystems: `qcommon` (networking/messaging), the server frame loop (`sv_main.c`/`sv_snapshot.c`), the Game VM (`gvm`), and the out-of-band authorize infrastructure at ID's remote server. Every client's journey from raw UDP packet to active game participant is controlled here. No other server file touches the client lifecycle so broadly — `sv_world.c` and `sv_bot.c` are narrower specialists, while `sv_snapshot.c` only handles outbound state.

## Key Cross-References

### Incoming (who depends on this file)

- **`sv_main.c`** — OOB packet dispatcher calls `SV_GetChallenge`, `SV_AuthorizeIpPacket`, `SV_DirectConnect`; also calls `SV_ExecuteClientMessage` per client per frame via `SV_PacketEvent`.
- **`sv_snapshot.c`** — calls `SV_WriteDownloadToClient` during snapshot generation for clients in download state; this is the only place download blocks are actually written out.
- **`sv_init.c`** — calls `SV_DropClient` during map changes and server restarts to flush active clients; `SV_SendClientGameState` is called here for reconnecting clients after map restart.
- **`sv_ccmds.c`** — operator commands like `clientkick` route through `SV_DropClient`.
- **`sv_bot.c`** — bot lifecycle calls `SV_DropClient` when a bot slot is released.

### Outgoing (what this file depends on)

- **Game VM (`gvm`)** via `VM_Call`: `GAME_CLIENT_CONNECT`, `GAME_CLIENT_DISCONNECT`, `GAME_CLIENT_BEGIN`, `GAME_CLIENT_USERINFO_CHANGED`, `GAME_CLIENT_THINK` — this file is the primary exerciser of the game VM's client-facing API surface.
- **`qcommon` networking**: `Netchan_Setup`, `MSG_*` (read/write/init), `NET_OutOfBandPrint`, `NET_CompareAdr`, `NET_StringToAdr`, `Huffman`-compressed bitstream ops.
- **`qcommon` filesystem**: `FS_SV_FOpenFileRead`, `FS_Read`, `FS_FCloseFile`, `FS_idPak`, `FS_LoadedPakPureChecksums`, `FS_ReferencedPakPureChecksums` — the download and pure-server validation paths.
- **`sv_main.c`**: `SV_Heartbeat_f` (called on first/last client connect/disconnect), `sv_*` cvars (`sv_maxclients`, `sv_minPing`, `sv_maxPing`, `sv_privateClients`, `sv_strictAuth`, etc.), and the global `svs`/`sv` state.
- **`sv_snapshot.c`**: `SV_SendMessageToClient`, `SV_UpdateServerCommandsToClient` for gamestate transmission.

## Design Patterns & Rationale

**Challenge-response as DoS mitigation**: The `svs.challenges[MAX_CHALLENGES]` ring buffer with oldest-eviction implements a stateless challenge protocol. The challenge value `(rand() << 16) ^ rand() ^ svs.time` combines two 15-bit randoms and a timestamp, providing ~45 bits of unpredictability — modest by modern standards but sufficient for 1999 UDP spoofing scenarios.

**Temporary client struct (`MAC_STATIC client_t temp`)**: `SV_DirectConnect` builds the new client into a stack-allocated (or `MAC_STATIC` for Mac Classic's limited stack) `client_t`, then atomically `*newcl = temp` copies it into the real slot. This is a commit pattern — if anything fails before `gotnewcl`, no slot is corrupted. The `MAC_STATIC` annotation (`static` on Mac, nothing elsewhere) is a portability hack visible across the codebase.

**Structured goto (`gotnewcl`)**: Used deliberately for reconnect vs. new-connect paths that converge at the same initialization block. This was idiomatic in id's codebase as an alternative to deeply nested `if`/`else` — modern code would use an early-return helper instead.

**`SV_VerifyPaks_f`'s `while(bGood) { break; }` pattern**: This is a labeled-block emulation — a structured way to write code that can bail out of a long validation sequence without deeply nested conditions or `goto`. Common in id's server code where functions grew organically.

**Delta compression keyed by gameplay state**: The usercmd delta key (`checksumFeed ^ messageAcknowledge ^ hash(reliableCommands)`) ties the decompression key to session state that a passive observer cannot trivially replay, cheaply authenticating the command stream without a separate HMAC.

## Data Flow Through This File

```
UDP socket (raw bytes)
    └─► sv_main.c OOB dispatcher
            ├─► SV_GetChallenge         → writes svs.challenges[]
            │                           → NET_OutOfBandPrint (challengeResponse / getIpAuthorize)
            ├─► SV_AuthorizeIpPacket    → reads svs.challenges[], sends challengeResponse or rejection
            └─► SV_DirectConnect        → reads svs.challenges[], allocates svs.clients[] slot
                                        → VM_Call(GAME_CLIENT_CONNECT)
                                        → Netchan_Setup → newcl->netchan
                                        → NET_OutOfBandPrint("connectResponse")
                                        → newcl->state = CS_CONNECTED

Per-frame (sv_main.c → SV_PacketEvent):
    SV_ExecuteClientMessage(cl, msg)
        ├─ reads message/reliable ACKs → cl->messageAcknowledge, cl->reliableAcknowledge
        ├─ SV_ClientCommand → dispatches ucmds[] table
        │       ├─ SV_VerifyPaks_f  → reads FS pak checksums → cl->pureAuthentic
        │       ├─ SV_Disconnect_f  → SV_DropClient
        │       └─ SV_DownloadCommand_f → sets cl->downloadName
        └─ SV_UserMove → MSG_ReadDeltaUsercmdKey → SV_ClientThink(VM_Call GAME_CLIENT_THINK)

sv_snapshot.c → SV_WriteDownloadToClient(cl, msg):
    reads cl->download* → FS_Read → writes svc_download blocks → msg
```

State transitions: `CS_FREE → CS_CONNECTED` (SV_DirectConnect) → `CS_PRIMED` (SV_SendClientGameState) → `CS_ACTIVE` (SV_UserMove/SV_ClientEnterWorld) → `CS_ZOMBIE` (SV_DropClient).

## Learning Notes

- **VM boundary discipline**: Every call crossing the engine/game boundary goes through `VM_Call(gvm, GAME_CLIENT_*)`. Return values are pointers in VM address space and **must** be re-mapped with `VM_ExplicitArgPtr` before use in engine code (visible at the `denied` pointer after `GAME_CLIENT_CONNECT`). This is a common footgun in id's VM sandbox that this file demonstrates correctly.

- **Pure server enforcement era**: The `SV_VerifyPaks_f` logic is a representative example of late-1990s integrity enforcement: checksums of loaded pk3 files, verified against a server-held list. Modern engines use signed update channels; Q3 relied on the ID authorize server as the trust anchor, which is why the `AUTHORIZE_TIMEOUT` fallback ("let them in anyway") exists.

- **No ECS here**: Entity association (`newcl->gentity = SV_GentityNum(clientNum)`) is a direct index into a flat `sharedEntity_t` array — classic flat entity array architecture, predating component systems. The `clientNum` is the canonical identifier bridging the network layer and the game layer.

- **Rate-limited reliable download over UDP**: The sliding-window download in `SV_WriteDownloadToClient` implements a poor-man's reliable transfer inside an unreliable UDP snapshot message. Blocks are re-sent until ACKed. This predates QUIC and was necessary because in 1999 there was no HTTP-based asset delivery pipeline.

- **`MAC_STATIC`**: A cross-platform stack-size workaround. Worth understanding when reading any function that declares large local structs — it signals that the developer knew the struct was too large for Mac Classic's 8 KB stack.

## Potential Issues

- **`sprintf` with user-controlled `r` (authorize reason string)**: In `SV_AuthorizeIpPacket`, `sprintf(ret, "print\n%s\n", r)` where `r = Cmd_Argv(3)` comes from an authorize server packet. If the authorize server were spoofed or compromised, an oversized `r` would overflow the 1024-byte `ret` buffer. The 1999 threat model considered the authorize server trusted, but this would be a stack buffer overflow in modern analysis.

- **`svs.challenges` race**: `SV_GetChallenge` and `SV_AuthorizeIpPacket` both read/write `svs.challenges[]` without locking. In single-threaded Q3 this is safe, but any async/threaded port would need synchronization here.

- **Challenge entropy**: `(rand() << 16) ^ rand()` uses `rand()` which on many platforms has only 15 bits of state per call. The actual entropy of the challenge is ~30 bits XOR'd with `svs.time` — weak by 2026 standards but acceptable for 1999 LAN/modem play.
