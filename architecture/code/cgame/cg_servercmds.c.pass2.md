# code/cgame/cg_servercmds.c — Enhanced Analysis

## Architectural Role

This file is the **server-command dispatcher and state-update gateway** for the cgame VM. It consumes the reliable, sequenced command stream from the server (`svc_` messages) and translates them into cgame state mutations and subsystem invocations. It bridges the authoritative server simulation (running `gvm`/`game.qvm` code) with the client-side rendering, prediction, and UI layers. At snapshot boundaries, this dispatcher fires once per server frame, ensuring all state changes are processed before rendering the new snapshot.

## Key Cross-References

### Incoming (who depends on this file)

- **`cg_snapshot.c:CG_ProcessSnapshots`** — calls `CG_ExecuteNewServerCommands` at snapshot transition (the primary caller)
- **`cg_draw.c` / render loop** — calls `CG_PlayBufferedVoiceChats` every frame to dequeue voice chat ring buffer
- **`cg_main.c:CG_Init`** — calls `CG_ParseServerinfo` and `CG_SetConfigValues` at level load
- **Voice chat lookup chain** — `be_ai_chat.c` (game VM) calls `trap_Cvar_VariableStringBuffer` for voice file paths, which then calls back to cgame via voice chat dispatch

### Outgoing (what this file depends on)

- **Trap syscalls**: `trap_GetServerCommand`, `trap_GetGameState`, `trap_FS_FOpenFile/Read/Close` (voice file I/O), `trap_S_RegisterSound`, `trap_R_RegisterModel`, `trap_S_StartLocalSound`, `trap_Cvar_Set`
- **cgame internals**: `CG_StartMusic`, `CG_NewClientInfo`, `CG_BuildSpectatorString`, `CG_InitLocalEntities`, `CG_InitMarkPolys`, `CG_ClearParticles`, `CG_SetScoreSelection` (MISSIONPACK)
- **Global state**: reads/writes `cgs` (server-side game state snapshot), `cg` (per-frame client state), `voiceChatLists[]`, `voiceChatBuffer[]`, `sortedTeamPlayers[]`, `numSortedTeamPlayers`

## Design Patterns & Rationale

**1. Command Dispatch via String Tokenization**  
The `CG_ServerCommand` function is a classic VM boundary pattern: receive raw text, tokenize with `CG_Argv`, dispatch by string comparison. This is simpler than a binary opcode table and survives protocol version drift gracefully.

**2. Ring-Buffer Throttling for Voice Chat (MISSIONPACK)**  
The `voiceChatBuffer` and `CG_PlayBufferedVoiceChats` implement client-side flow control: multiple voice chats queued in one server frame are dequeued at 1 per second. This prevents network lag spikes from dumping 10 simultaneous sounds into the audio mixer. Modern engines would use priority queues; this era used brute-force rounding.

**3. Head-Model → Voice File Caching**  
`headModelVoiceChat` maps skin names to loaded `voiceChatList_t` indices, avoiding repeated file I/O. The cache is never invalidated, implying voice files are immutable per session and player models are assigned once at connect.

**4. Two-Phase State: `cgs` vs `cg`**  
`cgs` holds snapshot-to-snapshot invariants (team scores, serverinfo, client list); `cg` holds per-frame transient state (warmup count, entity arrays). This dispatcher updates both, maintaining the invariant that `cgs` changes are always synchronized across all clients for a given snapshot number.

## Data Flow Through This File

```
Server (game.qvm) → Reliable UDP channel → Client (qcommon/client)
    ↓
trap_GetServerCommand() dequeue sequence N
    ↓
CG_ServerCommand() tokenize & dispatch
    ├→ "cs" (config string)
    │   └→ CG_ConfigStringModified() 
    │       ├→ Re-register assets (models, sounds)
    │       ├→ Update clientinfo[], scores, flags
    │       └→ Trigger events (music, sounds)
    ├→ "vchat" / "vtchat" 
    │   └→ CG_VoiceChat()
    │       ├→ Load/cache head model voice file
    │       └→ CG_AddBufferedVoiceChat() (enqueue ring)
    ├→ "scores"
    │   └→ CG_ParseScores() (populate cg.scores[])
    ├→ "tinfo"
    │   └→ CG_ParseTeamInfo() (populate sortedTeamPlayers[])
    └→ ... (cp, print, chat, tchat, map_restart, etc.)
    
Per-frame dequeue:
    CG_PlayBufferedVoiceChats()
    └→ Dequeue one buffered voice chat per second
        ├→ trap_S_StartLocalSound()
        └→ Print to team chat if applicable
```

## Learning Notes

- **Event-driven VM boundary**: Unlike stateful pull-based APIs (e.g., `get_server_state()`), this engine uses reliable push commands. The server is the authority; the client *reacts* to server announcements. This design predates HTTP2 and WebSockets but is conceptually similar.
- **Voice chat as a solved problem (circa 2005)**: Rather than stream audio, the game preloads `.voice` files per-player-model and plays discrete samples on demand with text overlay. This avoided the complexity of VoIP codecs and NAT traversal.
- **Ifdefs for game mode evolution**: The `#ifdef MISSIONPACK` markers trace the evolution from base Q3A (no team voice) → Team Arena (voice orders, ring buffer, team task synchronization).
- **Ring buffer as poor-man's priority queue**: The voice chat dequeue is strictly FIFO at 1-per-second; no prioritization. A modern engine would sort by importance or use adaptive throttling.
- **No transaction semantics**: If a server command is lost (should never happen on reliable channel), the client state may diverge from the server. The full snapshot is the recovery mechanism.

## Potential Issues

1. **Voice file I/O on hot path**: `CG_VoiceChatListForClient` calls `CG_HeadModelVoiceChats` → `trap_FS_FOpenFile` during `CG_VoiceChat` dispatch. A cache miss could stall the command dispatcher. Network lag hiding this latency is assumed.

2. **Buffer overrun in `CG_ParseVoiceChats`**: The voice file parser (`COM_ParseExt` loop) does not validate token lengths against `MAX_CHATSIZE` (64 bytes). A malformed `.voice` file with overly long chat strings could overflow `voiceChat_t.chats[*]`.

3. **Team chat ring buffer index wraparound**: `cgs.teamChatPos` increments unbounded; the modulo wraparound in `CG_AddToTeamChat` assumes `chatHeight` never changes. If `cg_teamChatHeight` cvar is updated mid-session, the buffer math could produce stale/corrupted messages.

4. **No cache invalidation on disconnect**: `headModelVoiceChat` entries are never pruned; a player model renamed/swapped at runtime would reuse the old cache entry, potentially playing the wrong voice file.

5. **No bounds on `numValidOrders`**: The `validOrders` table is hardcoded; if a server tries to send a voice order outside the table, `CG_ValidOrder` returns -1 silently (MISSIONPACK path), but the caller may not handle it defensively.
