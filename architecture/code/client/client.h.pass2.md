# code/client/client.h — Enhanced Analysis

## Architectural Role

This file defines the **client-engine boundary layer** for Quake III's client-server architecture. It serves three critical roles: (1) establishes the snapshot-based protocol contract between server and client, encapsulating periodic server-state deltas; (2) defines the three-tier persistent state hierarchy (`clSnapshot_t` → `clientActive_t` → `clientConnection_t` → `clientStatic_t`) that survives at different time-scales (snapshot, gamestate, connection, application lifetime); and (3) orchestrates the frame loop bridging the monolithic qcommon core to three independent VM-hosted subsystems (cgame, ui, renderer). It is the exclusive authority over the client connection state machine (challenged → connected → spawned → active) and the protocol details (packet sequencing, reliable commands, demo recording, file downloads).

## Key Cross-References

### Incoming (who depends on this file)
- **All client modules** (`cl_main.c`, `cl_parse.c`, `cl_input.c`, `cl_keys.c`, `cl_console.c`, `cl_scrn.c`, `cl_cin.c`, `cl_cgame.c`, `cl_ui.c`, `cl_net_chan.c`) include `client.h` and read/write the global `cl`, `clc`, `cls` state
- **Main engine loop** (`qcommon/common.c::Com_Frame`) invokes `CL_Frame` (declared here) and `SV_Frame` to drive both client and server each frame
- **Platform I/O layers** (`win32/win_input.c`, `unix/linux_joystick.c`) route input events into kbutton_t globals; (`win32/win_snd.c`, `unix/linux_snd.c`) read DMA mixer state from `snd_*.c` which consumes `clientActive_t`
- **Renderer** (`renderer/tr_*.c`) consumes `re` function pointers declared here and calls back into cgame for entity culling/rendering
- **cgame and ui VMs** call back into engine via `trap_*` syscall handlers, reading from `cl.snap`, `cl.gameState`, and writing to `cgameUserCmdValue`, `cgameSensitivity`

### Outgoing (what this file depends on)
- **qcommon subsystem**: `msg.c` (MSG_ReadBits/WriteBits), `net_chan.c` (Netchan_Transmit/Process), `vm.c` (VM_Call/Create), `files.c` (FS_*), `cmd.c` (Cbuf_Execute), `cvar.c` (Cvar_Get/Set)
- **Renderer**: `tr_public.h::refexport_t` contains the `re.Begin/End/Add*` function pointers for 2D/3D rendering
- **cgame/ui VMs**: `cg_public.h::cgameExport_t`, `ui_public.h::uiExport_t` define the exported vmMain entry points and syscall indices
- **Collision model**: `cm_public.h` (CM_ClusterPVS, CM_BoxTrace) used for entity movement prediction and PVS culling
- **Sound system** (`snd_public.h`, `snd_local.h`): software mixer, asset loading, streaming; integrated into client frame loop

## Design Patterns & Rationale

**Three-tier state hierarchy** (`clientStatic_t` ⊃ `clientConnection_t` ⊃ `clientActive_t`):
- **clientStatic_t** — never zeroed; persists across disconnects, map loads, UI transitions. Holds connection state machine, server browser lists, render config, subsystem-started flags.
- **clientConnection_t** — zeroed on disconnect; holds Netchan state, reliable command queues, demo/download state, per-connection challenge/checksumFeed. Exclusive owner of sequenced UDP protocol.
- **clientActive_t** — wiped on every new gameState_t (map load); holds snapshot ring, entity baselines, parsed entities, user commands, view angles, time-delta tracking.
  
  *Rationale*: Allows fine-grained reset scope—a map change doesn't lose network sequence numbers, but does reset snapshot state.

**Snapshot ring buffer** (`clientActive_t.snapshots[PACKET_BACKUP]`, `.parseEntities[MAX_PARSE_ENTITIES]`):
- Ring of 32 snapshots maintains history for delta decompression; delta-compressed messages reference older snapshots as deltas.
- parseEntitiesNum index is intentionally **not anded off**, allowing wraparound detection (checked by callers in `cl_parse.c`).
  
  *Rationale*: Handles out-of-order and dropped packets; classic delta-compression sliding-window pattern.

**kbutton_t input polling**:
- Tracks two simultaneously-held key codes, down timestamp, msec duration in current frame, active flag, and wasPressed sticky flag.
- Polled each frame; not event-driven. Decouples input sampling rate from frame rate.
  
  *Rationale*: Pre-modern design; allows smooth input even if frame rate stutters. Modern engines use event-driven input with message queues.

**Demo recording as first-class connection state**:
- `demorecording`, `demoplaying`, `demowaiting` flags in `clientConnection_t` indicate demo state within the same state machine as live server connection.
  
  *Rationale*: Demos are treated as a degenerate server; same snapshot/entity parsing logic applies. Simplifies code but conflates demo and live connection handling.

## Data Flow Through This File

**Inbound snapshot path:**
```
Network → CL_ReadPackets() 
  → CL_Netchan_Process() [Huffman decompression]
  → CL_ParseServerMessage(msg_t *msg)
    → dispatches on svc_* opcode
    → svc_snapshot: updates cl.snapshots[], cl.parseEntities[]
    → svc_serverCommand: executes cl.serverCommands[]
    → svc_configstring: updates cl.gameState
  → CL_AdjustTimeDelta() corrects cl.serverTimeDelta for network drift
```

**Outbound command path:**
```
Input events (key/mouse) → kbutton_t globals (in_mlook, in_strafe, etc.)
  → CL_SendCmd() [each frame]
    → CL_KeyState(kbutton_t *) reads current button state
    → assembles usercmd_t from button deltas + view angles
    → CL_WritePacket() 
      → serializes usercmd_t, reliable command queue to msg_t
      → CL_Netchan_Transmit() [Huffman compression]
      → stores timing metadata in cl.outPackets[outPacketIndex]
  → Server receives, executes, acknowledges
```

**Rendering path:**
```
cl.snapshots[cl.snap.messageNum] (current snapshot)
  → CL_CGameRendering() calls cgvm vmMain(CG_PAINT_SNAPSHOT)
    → cgame consumes cl.snap, cl.parseEntities[] 
    → predicts Pmove on unacknowledged cmds
    → populates scene with entities, calls re.AddRefEntity()
  → SCR_UpdateScreen() calls re.BeginFrame/EndFrame
    → executes all re.Add* commands as render queue
```

## Learning Notes

**Pull-based snapshot architecture** (vs. event-based):
- Server sends periodic snapshots (max `cl_maxpackets` times per second); client must interpolate/extrapolate between snapshots.
- Contrast to modern engines (Unreal, Unity) which use client-side prediction + authoritative server corrections via events.
- Snapshot size is fixed per-packet; network bandwidth is predictable but latency jitter causes interpolation artifacts.

**Deterministic client-side prediction**:
- cgame and game VMs share identical `bg_pmove.c` to run Pmove locally; client predicts unacknowledged commands, then corrects when server snapshot arrives.
- This pattern (predict-correct) is **not** visible in this header but is a fundamental design assumption.

**Persistent server browser state**:
- Server lists (localServers, globalServers, favoriteServers, mplayerServers) live in `clientStatic_t`, not in a UI module. This predates modern server-browser plugins.
- Implies tight coupling between engine and server discovery; difficult to swap.

**Demo as first-class primitive**:
- Unlike modern engines where demos are often post-processing captures, Q3's demo system is architectural—reuses the entire snapshot/parsing pipeline.
- Enables detailed bot replay and server-state inspection but complicates connection logic (see `demowaiting`, `demorecording` flags).

**XOR obfuscation of packets** (not visible in header, but `cl_net_chan.c`):
- Challenge-derived rolling XOR key applied to outgoing/incoming messages for trivial obfuscation (not encryption).
- Predates modern TLS; purely anti-sniffing, not anti-tampering.

## Potential Issues

- **Ring buffer wraparound**: `cl.parseEntitiesNum` is not masked; if logic doesn't account for wraparound, negative indices or off-by-one errors may occur when comparing against baseline.
- **Hardcoded limits**: MAX_PARSE_ENTITIES=2048, PACKET_BACKUP=32, MAX_RELIABLE_COMMANDS=64 limit concurrent entity/command counts; exceeding them causes silent truncation or parsing errors.
- **Time-delta drift**: `cl.serverTimeDelta` is adjusted each frame to correct for network latency variance, but if drift is large, interpolation can stutter or jump discontinuously.
- **Download resumption**: `downloadBlock` tracking is manual; if a large file transfer times out mid-download, resume logic must correctly seek to the right block offset, or corruption/retry loops result.
- **Pure server validation**: `checksumFeed` from server is used by `pak` signature verification; a single bit flip in checksumFeed causes all pak validation to fail and disconnect the client silently.
