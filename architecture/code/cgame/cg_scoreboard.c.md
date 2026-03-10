# code/cgame/cg_scoreboard.c

## File Purpose
Renders the in-game scoreboard overlay for Quake III Arena, including both the standard mid-game scoreboard and the oversized tournament intermission scoreboard. It handles FFA, team, and spectator layouts with fade animations.

## Core Responsibilities
- Draw per-client score rows with bot icons, player heads, flag indicators, and score/ping/time/name text
- Handle adaptive layout switching between normal and interleaved (compact) modes based on player count
- Render ranked team scoreboards in correct lead order (leading team drawn first)
- Display killer name, current rank/score string, and team score comparison at top of screen
- Draw scoreboard column headers (score/ping/time/name icons)
- Render the full-screen tournament scoreboard with giant text for MOTD, server time, and player scores
- Ensure the local client is always visible, appending their row at the bottom if scrolled off

## Key Types / Data Structures

| Name | Kind | Purpose |
|------|------|---------|
| `score_t` | struct (from `cg_local.h`) | Per-client score data: client index, score, ping, time, powerups, team |
| `clientInfo_t` | struct (from `cg_local.h`) | Client metadata: name, team, botSkill, handicap, wins/losses, powerups, model handles |
| `cg_t` | struct (from `cg_local.h`) | Global client game state; holds `scores[]`, `numScores`, `teamScores[]`, `killerName`, etc. |
| `cgs_t` | struct (from `cg_local.h`) | Static client game state; holds `gametype`, `clientinfo[]`, `media`, `maxclients` |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|------|------|-------|---------|
| `localClient` | `static qboolean` | file-static | Tracks whether the local client's row was drawn in the main list; if not, draws it pinned at the bottom |

## Key Functions / Methods

### CG_DrawClientScore
- **Signature:** `static void CG_DrawClientScore(int y, score_t *score, float *color, float fade, qboolean largeFormat)`
- **Purpose:** Renders a single player's row on the scoreboard.
- **Inputs:** `y` — vertical pixel position; `score` — score record; `color` — RGBA for text; `fade` — alpha multiplier; `largeFormat` — normal vs. interleaved height mode
- **Outputs/Return:** void
- **Side effects:** Sets `localClient = qtrue` if the row belongs to the local player. Draws to screen via `CG_DrawPic`, `CG_DrawHead`, `CG_DrawFlagModel`, `CG_FillRect`, `CG_DrawBigString`, `CG_DrawSmallStringColor`.
- **Calls:** `CG_DrawFlagModel`, `CG_DrawPic`, `CG_DrawSmallStringColor`, `CG_DrawHead`, `Com_sprintf`, `CG_FillRect`, `CG_DrawBigString`, `CG_DrawBigStringColor`
- **Notes:** Validates `score->client` bounds before dereferencing. Highlight color for local player encodes rank (0=blue, 1=red, 2=yellow, else gray). The `MISSIONPACK` block conditionally draws team task icons (offense/defense).

### CG_TeamScoreboard
- **Signature:** `static int CG_TeamScoreboard(int y, team_t team, float fade, int maxClients, int lineHeight)`
- **Purpose:** Iterates `cg.scores[]` and calls `CG_DrawClientScore` for each client on the specified team, up to `maxClients`.
- **Inputs:** `y` — starting Y; `team` — team filter; `fade`; `maxClients` — cap; `lineHeight` — row height constant
- **Outputs/Return:** `int` — number of clients drawn
- **Side effects:** Indirect screen draws via `CG_DrawClientScore`.
- **Calls:** `CG_DrawClientScore`
- **Notes:** Passes `largeFormat = (lineHeight == SB_NORMAL_HEIGHT)`.

### CG_DrawOldScoreboard
- **Signature:** `qboolean CG_DrawOldScoreboard(void)`
- **Purpose:** Main entry point for the in-game scoreboard. Handles fade logic, header text, column icons, team/FFA layout, and deferred player loading.
- **Inputs:** None (reads from `cg`, `cgs`)
- **Outputs/Return:** `qboolean` — `qtrue` if scoreboard was drawn; `qfalse` if suppressed (paused, single-player intermission, etc.)
- **Side effects:** Calls `CG_LoadDeferredPlayers()` after 10 frames of scoreboard display. Resets `cg.killerName` and `cg.deferredPlayerLoading` on early exit. Draws to screen.
- **Calls:** `CG_FadeColor`, `CG_DrawStrlen`, `CG_DrawBigString`, `CG_PlaceString`, `CG_DrawPic`, `CG_TeamScoreboard`, `CG_DrawTeamBackground`, `CG_DrawClientScore`, `CG_LoadDeferredPlayers`
- **Notes:** Adaptive layout: if `cg.numScores > SB_MAXCLIENTS_NORMAL`, switches to interleaved 16px rows. Leading team always drawn first. Local client pinned at bottom if `localClient` remains `qfalse`.

### CG_CenterGiantLine
- **Signature:** `static void CG_CenterGiantLine(float y, const char *string)`
- **Purpose:** Draws a centered string using giant (32×48) characters.
- **Inputs:** `y` — screen row; `string` — text
- **Outputs/Return:** void
- **Side effects:** Screen draw via `CG_DrawStringExt`.
- **Calls:** `CG_DrawStrlen`, `CG_DrawStringExt`

### CG_DrawOldTourneyScoreboard
- **Signature:** `void CG_DrawOldTourneyScoreboard(void)`
- **Purpose:** Renders the full-screen tournament/intermission scoreboard with MOTD, server clock, and giant-font player/team scores.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Sends a `"score"` client command every 2 seconds to refresh scores. Fills entire screen black. Screen draws.
- **Calls:** `trap_SendClientCommand`, `CG_FillRect`, `CG_ConfigString`, `CG_CenterGiantLine`, `CG_DrawStringExt`
- **Notes:** For FFA, iterates all `MAX_CLIENTS` slots checking `infoValid` and `team == TEAM_FREE`.

## Control Flow Notes
- `CG_DrawOldScoreboard` is called from the 2D draw pass (`CG_Draw2D` / `CG_DrawActive`) each frame when the scoreboard key is held or during death/intermission.
- `CG_DrawOldTourneyScoreboard` is called specifically during `GT_TOURNAMENT` intermission.
- Neither function participates in init or shutdown; they are pure per-frame render calls.

## External Dependencies
- `cg_local.h` — all shared cgame types, globals (`cg`, `cgs`), and function declarations
- **Defined elsewhere:** `CG_DrawFlagModel`, `CG_DrawPic`, `CG_DrawHead`, `CG_FillRect`, `CG_DrawBigString`, `CG_DrawBigStringColor`, `CG_DrawSmallStringColor`, `CG_DrawStringExt`, `CG_DrawStrlen`, `CG_FadeColor`, `CG_PlaceString`, `CG_DrawTeamBackground`, `CG_LoadDeferredPlayers`, `CG_ConfigString`, `trap_SendClientCommand`, `Com_Printf`, `Com_sprintf`
- Constants `SB_NORMAL_HEIGHT`, `SB_INTER_HEIGHT`, `SB_MAXCLIENTS_NORMAL`, `SB_MAXCLIENTS_INTER`, `SB_SCORELINE_X`, etc. are all `#define`d locally in this file.
