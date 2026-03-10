# code/cgame/cg_scoreboard.c — Enhanced Analysis

## Architectural Role

This file implements the in-game and tournament scoreboards—critical HUD composition components within the cgame VM's per-frame 2D rendering pipeline. It consumes server-delivered snapshot data (`score_t` arrays, `playerState_t`, `clientInfo_t`) to compose two distinct visual presentations: the in-game overlay scoreboard (called from `CG_Draw2D` during gameplay/death/intermission) and the full-screen tournament intermission scoreboard. The file bridges snapshot data-structures (populated by the server, synchronized by `cg_snapshot.c`) with renderer syscalls, demonstrating cgame's role as the bridge between server authority and client presentation.

## Key Cross-References

### Incoming (who depends on this file)
- **cgame render pipeline:** `CG_Draw2D` (in `cg_draw.c` or equivalent) calls `CG_DrawOldScoreboard()` and/or `CG_DrawOldTourneyScoreboard()` conditionally during the per-frame screen composition
- **cgame frame loop:** The condition checks (`cg.showScores`, `cg.predictedPlayerState.pm_type == PM_INTERMISSION`, `cg.killerName`) are set by `cg_main.c` and `cg_snapshot.c`
- **Server state:** Indirectly depends on `cg.scores[]` populated by `cg_snapshot.c` from server `entityState_t` delta-compression

### Outgoing (what this file depends on)
- **Renderer syscalls (trap_R_*):** All visible drawing (`CG_DrawPic`, `CG_DrawHead`, `CG_DrawBigString`, `CG_FillRect`, `CG_DrawFlagModel`) eventually invoke renderer DLL via `refimport_t ri` syscalls; the cgame wrappers hide the syscall boundary
- **cgame intrinsics:** `CG_FadeColor`, `CG_PlaceString`, `CG_DrawStrlen`, `CG_ConfigString`, `CG_LoadDeferredPlayers` (asset streaming), `CG_DrawTeamBackground`
- **Global cgame state:** `cg` (dynamic: scores[], killerName, showScores, warmup) and `cgs` (static: gametype, clientinfo[], media shaders/models, maxclients)
- **Server connection:** `trap_SendClientCommand("score")` in tournament scoreboard refreshes score snapshots every 2 seconds
- **Math/utilities:** `Com_sprintf`, `q_math.c` constants (vector operations in `CG_DrawHead`)

## Design Patterns & Rationale

**Layout Adaptation (Normal vs. Interleaved):** The switch between `SB_NORMAL_HEIGHT` (40px) and `SB_INTER_HEIGHT` (16px) modes when `cg.numScores > SB_MAXCLIENTS_NORMAL` is a space-efficient response to variable player counts. This avoids clipping or off-screen rows while keeping the scoreboard visible. The thresholds are compile-time constants, reflecting Quake III's fixed 640×480 virtual resolution.

**File-Static `localClient` Flag:** This boolean tracks whether the local player's row was drawn in the main loop. If not (scrolled off), the row is pinned at the bottom. While inelegant (mutable global state), it solves the UX problem of the local player always being visible without re-scanning the score list or duplicating rendering logic. The pattern assumes `CG_DrawOldScoreboard` is called exactly once per frame.

**Leading-Team-First Ordering (Team Modes):** In `GT_TEAM` modes, the scoreboard is redrawn with the leading team first (`cg.teamScores[0] >= cg.teamScores[1]`), immediately followed by backgrounds and the trailing team. This emphasizes the winning team visually and ensures consistent reading order.

**Deferred Player Loading:** The `cg.deferredPlayerLoading` counter and `CG_LoadDeferredPlayers()` call every 10+ frames decouple asset streaming from the render frame, preventing frame-time stalls when many client models are first displayed on the scoreboard.

## Data Flow Through This File

1. **Ingress:** `cg.scores[]` (snapshot-synchronized per frame), `cg.snap->ps.clientNum` (local player ID), `cg.killerName` (set by death sequence), `cg.warmup` / `cg.showScores` (state flags), `cgs.gametype` (game mode determines layout)
2. **Transformation:**
   - Check visibility conditions (paused, single-player intermission, warmup, fade state)
   - Compute fade color and alpha based on `CG_FadeColor(cg.scoreFadeTime, FADE_TIME)`
   - Select layout mode (interleaved vs. normal) based on player count
   - Iterate `cg.scores[]`, filter by team (via `CG_TeamScoreboard`), build formatted score strings
   - Calculate highlight color for local player based on rank tier
3. **Egress:** Screen-space draw commands via `CG_DrawPic` (icons, headers, flag models), `CG_DrawHead` (player face models), `CG_DrawBigString` (score text), `CG_FillRect` (highlight background)

## Learning Notes

**Immediate-Mode GUI Pattern:** This file exemplifies immediate-mode rendering—no retained state, pure function of `cg` globals and elapsed time. Every call to `CG_DrawOldScoreboard()` recomputes the full scoreboard from scratch, making it simple but requiring the `localClient` flag hack to ensure pinning.

**Tournament Scoreboard as Special Case:** The oversized tournament scoreboard (`CG_DrawOldTourneyScoreboard`) is a distinct full-screen overlay using giant (32×48px) text. It fills the entire screen black and periodically sends a `"score"` command to refresh server data—a polling pattern suited to intermission where real-time updates are less critical.

**VirtualUI Coordinate Space:** All layout uses pixel offsets in 640×480 virtual space (e.g., `SB_SCORELINE_X = 112`), independent of actual display resolution. The renderer scales this to the physical framebuffer at syscall time.

**Rank-Based Coloring:** The local player's highlight uses rank to select a color (rank 0→blue, 1→red, 2→yellow, else gray). This design couples presentation to game logic (`PERS_RANK`), embedding game knowledge into the UI.

**Asset Deferred Loading:** The pattern of deferring `CG_LoadDeferredPlayers()` to every 10+ frames reflects a trade-off: first scoreboard appearance may show placeholder models, but frame hitches are minimized.

**MISSIONPACK Conditional Blocks:** Lines with `#ifdef MISSIONPACK` (team task icons) indicate this codebase supports multiple game variants (base Q3A vs. Team Arena), suggesting a build-time multiplex architecture.

## Potential Issues

- **`localClient` Mutation:** The static flag is reset to `qfalse` at start of `CG_DrawOldScoreboard` but set to `qtrue` inside `CG_DrawClientScore`. If the latter is called from multiple contexts or the function is called multiple times per frame, the flag could enter an inconsistent state. A return value from `CG_TeamScoreboard` indicating "did we draw the local client" would be safer.

- **Bounds Validation:** `score->client` is checked in `CG_DrawClientScore`, but `cg.scores[i].client` is never validated in the loop—a corrupt score array could cause out-of-bounds `cgs.clientinfo[]` access.

- **Fade Logic Coupling:** The complex fade conditions (showScores, PM_DEAD, PM_INTERMISSION, warmup, scoreFadeTime) are scattered across early returns, making the visibility contract implicit rather than explicit.
