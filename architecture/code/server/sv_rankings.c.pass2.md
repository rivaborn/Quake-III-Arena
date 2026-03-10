# code/server/sv_rankings.c — Enhanced Analysis

## Architectural Role

`sv_rankings.c` bridges the Quake III Arena **server** subsystem with an external **GRank (Global Rankings)** third-party library, providing async match and player stat reporting to a remote rankings service. It acts as a shim layer that translates frame-synchronous server operations into callback-driven async API calls, enforcing safe context lifecycle management via reference counting. This module is optional (gated by `sv_enableRankings` cvar) and decoupled from core server functionality, making it a clean, contained extension point for telemetry and competitive rank tracking.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/server/sv_main.c`** (frame loop): calls `SV_RankPoll()` each frame to drive GRank event queue; calls `SV_RankBegin()` at match start, `SV_RankEnd()` at match shutdown
- **`code/server/sv_client.c`** (client lifecycle): calls `SV_RankUserLogin()`, `SV_RankUserCreate()`, `SV_RankUserValidate()` on client connect/auth; calls `SV_RankUserLogout()` on disconnect
- **`code/server/sv_game.c`** (server-game boundary): calls `SV_RankReportInt()`, `SV_RankReportStr()` to submit gameplay stats (kills, deaths, flags, damage) during match
- **`code/game/g_rankings.c`** (game VM, if present): may call ranking-related trap functions; not visible in this snapshot but likely exists in Team Arena build
- **`code/server/server.h`** (public exports): declares all public `SV_Rank*` function prototypes; cvars `sv_enableRankings`, `sv_rankingsActive`, `sv_leagueName` referenced here
- **Configstring `CS_GRANK`** (line ~600): game engine writes encoded game ID here; cgame/UI may read it for display purposes

### Outgoing (what this file depends on)
- **`../rankings/1.0/gr/grapi.h`** (external GRank library): core async API: `GRankInit`, `GRankNewGameAsync`, `GRankUserLoginAsync`, `GRankUserCreateAsync`, `GRankJoinGameAsync`, `GRankPlayerValidate`, `GRankStartMatch`, `GRankReportInt`, `GRankReportStr`, `GRankSendReportsAsync`, `GRankCleanupAsync`, `GRankPoll`
- **`../rankings/1.0/gr/grlog.h`** (GRank logging): `GRankLogLevel`, `GRLOG_OFF`, `GRLOG_TRACE`
- **`code/qcommon`** globals and functions:
  - `Cvar_Set()`, `Cvar_VariableValue()`, `Cvar_VariableString()` for config state
  - `Z_Malloc()`, `Z_Free()` for `s_ranked_players` allocation
  - `SV_SetConfigstring()` to write `CS_GRANK` for broadcast to clients
  - `Com_DPrintf()` for debug logging
  - `Info_SetValueForKey()` for configstring manipulation
  - `LittleLong64()` for byte-order conversion (not visible in truncated file but inferred from encoding/decoding 64-bit IDs)

## Design Patterns & Rationale

**1. Callback-Driven Async Integration**
- GRank library provides no blocking API—all long-latency operations (network I/O, server registration, player auth) are async with user-supplied callbacks (`SV_RankNewGameCBF`, `SV_RankUserCBF`, etc.).
- `SV_RankPoll()` pumps the event loop each frame; callbacks execute in-frame when data arrives, guaranteeing single-threaded execution.
- **Rationale:** Avoids blocking the server frame loop on network I/O; allows graceful degradation if rankings service is slow/unreachable.

**2. Reference Counting for Context Lifecycle**
- `s_rankings_contexts` counts all live `GR_CONTEXT` handles (server + all player contexts). 
- `SV_RankCloseContext()` decrements; when count reaches zero: `s_ranked_players` is freed and logging is disabled.
- **Rationale:** Ensures cleanup only happens after all async operations complete (e.g., final `GRankCleanupAsync` callback fires before heap is freed). Prevents use-after-free and double-free bugs inherent to async callback chains.

**3. Per-Player Context Separation**
- Server has one global context (`s_server_context`); each authenticated player gets a dedicated context allocated during login/create.
- **Rationale:** GRank library likely binds context to logical session identity. Separating prevents cross-player state leakage and allows independent auth/match tracking per player.

**4. Encoding/Decoding Tokens Without Direct Library Integration**
- Custom 6-bit ASCII encoding (`s_ascii_encoding` table) allows game ID, player ID, and auth tokens to be serialized into printable strings safe for transmission via configstrings and network messages.
- `SV_RankAsciiEncode/Decode` with lazy-init inverse lookup avoids repeated table scans.
- **Rationale:** GRank library works with opaque binary blobs; encoding to ASCII ensures cross-platform compatibility in text-based game data serialization.

**5. Status Machine with Deferred Cleanup**
- Player status enum (`QGR_STATUS_*`) tracks: `NEW`, `PENDING`, `SPECTATOR`, `ACTIVE`, `NO_USER`, `TIMEOUT`, `ERROR`, etc.
- `final_status` field allows callback chains to queue a status update for after cleanup completes.
- **Rationale:** Callback ordering is strict (login → join game → validate); intermediate states must not block the next stage, but final error reporting must survive async cleanup.

## Data Flow Through This File

```
┌─────────────────────────────────────────────────────────────────┐
│ MATCH INIT (sv_main.c → SV_RankBegin)                            │
├─────────────────────────────────────────────────────────────────┤
│ • Cvar check: sv_enableRankings, sv_leagueName                  │
│ • GRankInit(1, gamekey) → server context                        │
│ • Allocate s_ranked_players[sv_maxclients]                      │
│ • GRankNewGameAsync → [frame] → SV_RankNewGameCBF              │
│ • [CBF SUCCESS] game_id, match handle, CS_GRANK written        │
│                   s_rankings_active = qtrue                     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PER-FRAME PUMP (sv_main.c → SV_RankPoll)                         │
├─────────────────────────────────────────────────────────────────┤
│ • GRankPoll() drains event queue                                │
│ • May fire SV_RankUserCBF, SV_RankJoinGameCBF, etc.            │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ PLAYER AUTH (sv_client.c → SV_RankUserLogin/Create)             │
├─────────────────────────────────────────────────────────────────┤
│ • GRankInit(0, gamekey) → per-player context                    │
│ • GRankUserLoginAsync/CreateAsync → [frame] → SV_RankUserCBF  │
│ • [CBF SUCCESS] GRankJoinGameAsync → [frame] → SV_RankJoinGameCBF │
│ • [CBF SUCCESS] save player_id, token, rank; SV_RankUserValidate │
│                   s_ranked_players[i].grank_status = ACTIVE    │
│ • [CBF FAILURE] set final_status, trigger SV_RankCloseContext  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ GAMEPLAY STATS (sv_game.c → SV_RankReportInt/Str)               │
├─────────────────────────────────────────────────────────────────┤
│ • GRankReportInt on both player contexts (if index2 active)    │
│ • GRankReportStr for string events (weapons, sprees, etc.)     │
│ • Accumulated in GRank library's event buffer                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ MATCH END (sv_main.c → SV_RankEnd)                               │
├─────────────────────────────────────────────────────────────────┤
│ • SV_RankUserLogout for each ACTIVE player                      │
│ • GRankSendReportsAsync → [frame] → SV_RankSendReportsCBF      │
│ • [CBF] GRankCleanupAsync → [frame] → SV_RankCleanupCBF        │
│ • [CBF] SV_RankCloseContext (all), free s_ranked_players       │
│ • s_rankings_active = qfalse                                    │
└─────────────────────────────────────────────────────────────────┘
```

## Learning Notes

1. **Callback chains are linearized, not nested:** Each callback initiates the next async operation in sequence, avoiding callback pyramid-of-doom. Modern async-await would flatten this further.

2. **Deferred initialization pattern:** Async tasks block until GRank reports success. Match doesn't "activate" (`s_rankings_active = qtrue`) until `SV_RankNewGameCBF` fires. Players aren't `ACTIVE` until `SV_RankJoinGameCBF` → validate completes. This serialization matches GRank's own API contract (new game must complete before players can join).

3. **Idiomatic to 2005-era networked game middleware:** External services were often wrapped via callback-based libraries (common in C game engines pre-async-await). The `LittleLong64` byte-swap for 64-bit IDs hints at Windows/console era when endianness was a cross-platform concern.

4. **No error recovery within callbacks:** If a callback fails (e.g., duplicate login), the response is to mark the player with a failure status and let cleanup proceed. No retry logic. This assumes GRank service is reliable or the match proceeds without rankings (graceful degradation).

5. **Configstring as IPC mechanism:** Game ID encoded into `CS_GRANK` ensures cgame and UI on the client can read rankings state without direct rankings library dependency. Clean separation of concerns.

6. **Polling not event-driven:** `SV_RankPoll()` is call-by-frame, not callback/interrupt-based. This design predates OS-level async I/O (epoll, IOCP) and reflects mid-2000s portable C patterns.

## Potential Issues

1. **No timeout on pending operations:** If GRank library hangs, there's no watchdog timer. A stalled callback chain could leave `s_rankings_contexts > 0` indefinitely, preventing proper shutdown.

2. **Reference count invariant not validated at shutdown:** If a callback fails to decrement the context count properly, `SV_RankQuit` will spin forever on the `SV_RankCheckInit()` poll loop.

3. **Encoding lookup table initialized lazily but not thread-safe:** `SV_RankAsciiDecode` inits an inverse lookup table on first use; if two threads call it simultaneously, a race condition could corrupt the table (not a problem if SMP render thread doesn't call ranking functions, which appears true).

4. **Truncated player name in ranked_player_t:** Fixed 32-byte `name` field; very long player names will be silently truncated without warning to the player.

5. **Hardcoded relative include path:** `#include "..\rankings\1.0\gr\grapi.h"` with backslash suggests Windows-centric development; this would fail on Unix without build-time path remapping.
