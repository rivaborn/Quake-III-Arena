# code/server/sv_rankings.c

## File Purpose
Implements the server-side interface to Id Software's Global Rankings (GRank) system, managing player authentication, match tracking, and stat reporting via an external rankings API. It bridges Quake III Arena's server loop with the asynchronous GRank library using callback-based operations.

## Core Responsibilities
- Initialize and shut down the GRank rankings session per game match
- Authenticate players via server-side login/create or client-side token validation
- Track per-player GRank contexts, match handles, and player IDs
- Submit integer and string stat reports for players/server during gameplay
- Handle asynchronous GRank callbacks for new game, login, join game, send reports, and cleanup
- Encode/decode player IDs and tokens using a custom 6-bit ASCII encoding scheme
- Manage context reference counting to safely free resources when all contexts close

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `ranked_player_t` | struct | Per-player rankings state: GRank context, match handle, player ID, token, status, rank, name |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `s_rankings_contexts` | `int` | static | Reference count of live GRank contexts (server + all players) |
| `s_rankings_active` | `qboolean` | static | Whether rankings are fully initialized and operational |
| `s_server_context` | `GR_CONTEXT` | static | GRank context for the server/match itself |
| `s_server_match` | `uint64_t` | static | Match handle for the server context |
| `s_rankings_game_key` | `char*` | static | Pointer to active game key string |
| `s_rankings_game_id` | `uint64_t` | static | Game ID returned by GRankNewGameAsync |
| `s_ranked_players` | `ranked_player_t*` | static | Heap-allocated array of per-client rankings state, size = `sv_maxclients` |
| `s_server_quitting` | `qboolean` | static | Quitting flag (declared but not actively used) |
| `s_ascii_encoding` | `const char[64]` | static | Custom 6-bit-to-ASCII lookup table |
| `SV_RankGameKey` | `char[64]` | static | Null-safe copy of game key passed to GRankInit |

## Key Functions / Methods

### SV_RankBegin
- **Signature:** `void SV_RankBegin( char *gamekey )`
- **Purpose:** Initializes rankings for a new match: inits GRank library, starts async new-game registration, allocates `s_ranked_players`.
- **Inputs:** `gamekey` — game mode key string (may be replaced with game-type-specific string)
- **Outputs/Return:** void
- **Side effects:** Allocates `s_ranked_players` via `Z_Malloc`; increments `s_rankings_contexts`; sets `sv_rankingsActive` cvar; calls `GRankNewGameAsync`
- **Calls:** `GRankLogLevel`, `GRankInit`, `GRankNewGameAsync`, `Cvar_Set`, `Z_Malloc`
- **Notes:** Early-outs if `sv_enableRankings == 0` or single-player mode. Commented-out pure-server check.

### SV_RankEnd
- **Signature:** `void SV_RankEnd( void )`
- **Purpose:** Ends the match: logs out all active players and sends server match reports asynchronously.
- **Inputs:** none
- **Outputs/Return:** void
- **Side effects:** Calls `SV_RankUserLogout` for active players; calls `GRankSendReportsAsync`; clears `s_rankings_active`
- **Calls:** `SV_RankUserLogout`, `GRankSendReportsAsync`, `Cvar_Set`

### SV_RankUserLogin / SV_RankUserCreate
- **Signature:** `void SV_RankUserLogin( int index, char* username, char* password )` / `void SV_RankUserCreate( int index, char* username, char* password, char* email )`
- **Purpose:** Initiates async server-side login or account creation for a player slot; allocates a dedicated GRank context.
- **Side effects:** `GRankInit` per player; increments `s_rankings_contexts`; sets player status to `QGR_STATUS_PENDING`
- **Calls:** `GRankInit`, `GRankUserLoginAsync` / `GRankUserCreateAsync`

### SV_RankUserValidate
- **Signature:** `qboolean SV_RankUserValidate( int index, const char* player_id, const char* key, int token_len, int rank, char* name )`
- **Purpose:** Validates a client-side pre-authenticated player using decoded player ID and token, or immediately approves bots (NULL id/key).
- **Outputs/Return:** `qtrue` if validated successfully
- **Side effects:** Decodes player ID/token; checks for duplicate logins; calls `GRankPlayerValidate`; sets player `grank_status`
- **Calls:** `GRankPlayerValidate`, `SV_RankDecodePlayerID`, `SV_RankDecodePlayerKey`

### SV_RankReportInt / SV_RankReportStr
- **Signature:** `void SV_RankReportInt( int index1, int index2, int key, int value, qboolean accum )` / `void SV_RankReportStr( int index1, int index2, int key, char* value )`
- **Purpose:** Submits a stat event (integer or string) to GRank, mirrored to both players' contexts when index2 is a valid active player.
- **Side effects:** Calls `GRankReportInt` / `GRankReportStr` once or twice
- **Notes:** index -1 means server context/no player.

### SV_RankNewGameCBF *(static)*
- **Signature:** `static void SV_RankNewGameCBF( GR_NEWGAME* gr_newgame, void* cbf_arg )`
- **Purpose:** Callback after async new-game registration. On success: saves game ID, encodes it into `CS_GRANK` config string, starts server match, sets `s_rankings_active = qtrue`.
- **Calls:** `GRankStartMatch`, `SV_SetConfigstring`, `Info_SetValueForKey`, `SV_RankEncodeGameID`

### SV_RankUserCBF *(static)*
- **Signature:** `static void SV_RankUserCBF( GR_LOGIN* gr_login, void* cbf_arg )`
- **Purpose:** Callback after login/create. On success chains to `GRankJoinGameAsync`; on failure sets `final_status` and triggers cleanup.

### SV_RankJoinGameCBF *(static)*
- **Signature:** `static void SV_RankJoinGameCBF( GR_JOINGAME* gr_joingame, void* cbf_arg )`
- **Purpose:** Callback after join-game. Saves player ID/token/rank; starts player match; calls `SV_RankUserValidate` with NULL id/key to finalize activation.

### SV_RankCloseContext *(static)*
- **Signature:** `static void SV_RankCloseContext( ranked_player_t* ranked_player )`
- **Purpose:** Decrements context refcount; resets player or server context fields. When count reaches zero: frees `s_ranked_players`, turns off logging.
- **Notes:** NULL argument means server context cleanup.

### SV_RankAsciiEncode / SV_RankAsciiDecode *(static)*
- **Purpose:** Custom base-64-like 6-bit encoding using `s_ascii_encoding` table. Encode packs 3 bytes → 4 chars; decode reverses. Uses lazy-initialized inverse lookup table.

## Control Flow Notes
- **Init:** `SV_RankBegin` is called at match start (from game/server init); rankings become active only after `SV_RankNewGameCBF` succeeds asynchronously.
- **Frame:** `SV_RankPoll` must be called each server frame to drive the async GRank event loop (`GRankPoll`).
- **Shutdown:** `SV_RankEnd` → `SV_RankSendReportsCBF` → `SV_RankCleanupCBF` → `SV_RankCloseContext` forms the async teardown chain. `SV_RankQuit` is a synchronous spin-poll fallback for forced exit.

## External Dependencies
- `server.h` — server types, cvars (`sv_maxclients`, `sv_enableRankings`, `sv_rankingsActive`), `SV_SetConfigstring`, `Z_Malloc`, `Z_Free`, `Cvar_Set`, `Cvar_VariableValue`, `Com_DPrintf`
- `../rankings/1.0/gr/grapi.h` — GRank API: `GRankInit`, `GRankNewGameAsync`, `GRankUserLoginAsync`, `GRankUserCreateAsync`, `GRankJoinGameAsync`, `GRankPlayerValidate`, `GRankSendReportsAsync`, `GRankCleanupAsync`, `GRankStartMatch`, `GRankReportInt`, `GRankReportStr`, `GRankPoll`; types `GR_CONTEXT`, `GR_STATUS`, `GR_PLAYER_TOKEN`, `GR_NEWGAME`, `GR_LOGIN`, `GR_JOINGAME`, `GR_MATCH`, `GR_INIT` — **defined in external rankings library, not in this file**
- `../rankings/1.0/gr/grlog.h` — `GRankLogLevel`, `GRLOG_OFF`, `GRLOG_TRACE` — **defined in external rankings library**
- `LittleLong64` — byte-order conversion for 64-bit values — **defined elsewhere in qcommon**
