# code/q3_ui/ui_gameinfo.c — Enhanced Analysis

## Architectural Role

This file is the **single-player progression and metadata hub** for the UI layer, bridging three key concerns: (1) **content metadata** (arenas and bot definitions loaded once at startup from filesystem), (2) **game state persistence** (single-player scores, awards, tier unlock state stored as cvars that survive session restarts), and (3) **tier progression enforcement** (special logic to keep SP level counts aligned to `ARENAS_PER_TIER` boundaries, enabling a structured campaign flow). It is the primary data source for `ui_spLevel.c` (tier/level selection menu), `ui_sppostgame.c` (postgame scoring), and award screens.

## Key Cross-References

### Incoming (who depends on this file)

- **`code/q3_ui/ui_main.c`** → calls `UI_InitGameinfo` during UI startup
- **`code/q3_ui/ui_spLevel.c`** → calls `UI_GetNumSPTiers`, `UI_GetArenaInfoByNumber`, `UI_GetArenaInfoByMap`, `UI_TierCompleted`, `UI_GetBestScore` to populate level selection menus and tier progression UI
- **`code/q3_ui/ui_sppostgame.c`** → calls `UI_SetBestScore`, `UI_TierCompleted` to record postgame scores and determine tier completion
- **`code/q3_ui/ui_spskill.c`** → calls cvar-dependent code to read `g_spSkill`
- **Menu initialization** → `UI_SPLevelMenu_ReInit` is called by `ui_spLevel.c` after `UI_SPUnlock_f` / `UI_SPUnlockMedals_f` console commands to refresh visible levels
- **Award screens** → query functions used to display medal/score tallies per skill level

### Outgoing (what this file depends on)

- **Engine VM interface (`trap_*` syscalls):**
  - `trap_Print` — debug/warning output
  - `trap_Cvar_Register`, `trap_Cvar_VariableValue`, `trap_Cvar_VariableStringBuffer`, `trap_Cvar_Set` — cvar lifecycle and read/write
  - `trap_FS_FOpenFile`, `trap_FS_Read`, `trap_FS_FCloseFile`, `trap_FS_GetFileList` — filesystem access for `.arena`, `.bot`, and `.txt` files
- **`qcommon` shared utilities** (linked into UI VM):
  - `COM_Parse`, `COM_ParseExt` — tokenize arena/bot definition files
  - `Info_SetValueForKey`, `Info_ValueForKey` — key/value pair manipulation (Q3's ubiquitous info-string format)
  - `Q_stricmp`, `Q_strncpyz` — string utilities
  - `va()` — varargs sprintf
- **`code/q3_ui/ui_atoms.c`** → reads/writes global `uis` state (e.g., `uis.demoversion`)
- **`code/q3_ui/ui_spLevel.c`** → `UI_SPLevelMenu_ReInit` called after unlock operations to refresh the level menu

## Design Patterns & Rationale

### 1. **Bump-Pointer Allocator (no deallocation)**
   - **Pattern:** Fixed 128 KB pool (`memoryPool`), linear allocation via `allocPoint`, 32-byte alignment padding
   - **Rationale:** Arena and bot metadata is loaded once at UI init and never freed during the session. A bump allocator is simpler and faster than a general-purpose heap for this use case. Alignment to 32-byte boundaries may be for cache efficiency (2000s-era optimization).
   - **Tradeoff:** Cannot unload maps mid-session without resetting the entire pool; memory fragmentation is irrelevant because there are no deallocations.

### 2. **Info-String Storage and Lookup**
   - **Pattern:** Each arena/bot is a single `Info_*`-formatted string (`\key\value\key\value...`) stored in the pool; queries iterate linearly over all strings to find matches by key value.
   - **Rationale:** Info strings are the canonical Q3 format for structured text data (used everywhere: serverinfo, userinfo, entity spawn parms). Single allocation per item + key/value API reduces memory fragmentation.
   - **Tradeoff:** O(n) linear search by `num` field in `UI_GetArenaInfoByNumber` is inefficient; a sorted index would speed up lookups, but the array size is small (≤256 arenas typical), so the overhead of maintaining an index is not justified.

### 3. **Single-Player Tier Alignment**
   - **Pattern:** After loading all arenas, the code counts SP arenas, discards any extra ones modulo `ARENAS_PER_TIER`, and reassigns `num` fields to impose ordering: `[0..singlePlayerNum-1]` = regular SP, `[singlePlayerNum..singlePlayerNum+specialNum-1]` = special (training/final), `[specialNum+specialNum..]` = other (CTF, etc.).
   - **Rationale:** The single-player campaign is structured as tiers; each tier must have the same number of levels for the menu to render correctly. Discarding levels ensures a regular grid.
   - **Tradeoff:** Some content is silently ignored; a stricter approach (error on misalignment) would catch configuration bugs but would break with maps that don't fit the tier structure.

### 4. **Cvar-Based Persistence**
   - **Pattern:** Single-player scores are stored as info strings in `g_spScores1` through `g_spScores5` cvars (one per skill level), with entries like `\l0\3` meaning "level 0 best score is 3rd place". Awards are in `g_spAwards0` through `g_spAwards5`. Tier unlock state is in `g_spVideos`.
   - **Rationale:** Cvars are persistent across session restarts (engine writes them to `q3config.cfg`); using them as a database avoids the need for a separate binary save-file format. Each skill level is isolated, so per-level per-skill querying is natural.
   - **Tradeoff:** Info-string format is not type-safe and requires manual parsing; a structured binary format would be safer but less human-readable for debugging.

## Data Flow Through This File

1. **Initialization Phase** (`UI_InitGameinfo`):
   - Zero the pool, load `scripts/arenas.txt` + all `*.arena` files into pool
   - Load `scripts/bots.txt` + all `*.bot` files into pool
   - Assign linear `num` indices to all arenas
   - Separate SP vs. special vs. other arenas; trim SP count to tier alignment
   - Reassign `num` indices so SP progression is contiguous

2. **Query Phase** (during menu navigation):
   - `UI_GetArenaInfoByNumber(n)` → lookup arena by tier-visible index
   - `UI_GetBestScore(level)` → read `g_spScores1..5` cvars, return best placement + skill
   - `UI_SetBestScore(level, score)` → write placement to the appropriate `g_spScoresX` cvar
   - `UI_TierCompleted(levelWon)` → check if all levels in this tier are finished at rank 1

3. **Unlock Phase** (cheat/debug commands):
   - `UI_SPUnlock_f` / `UI_SPUnlockMedals_f` write to `g_spVideos` cvar to unlock tier cinematics
   - Call `UI_SPLevelMenu_ReInit` to refresh the level menu display

## Learning Notes

### Q3-Idiomatic Patterns
- **Info strings** (`key\value\key\value`) are the canonical lightweight structured-text format throughout id Tech 3. Contrast with modern engines using JSON, YAML, or binary serialization.
- **Bump-pointer allocators** were common in late-90s/early-2000s game engines for predictable, cache-efficient allocation of static data.
- **Cvar-based persistence** is characteristic of engines with a strong REPL-like console heritage; it trades type safety and performance for human debuggability and minimal save-file infrastructure.

### Design Philosophy
- **Minimal runtime state:** Nearly all game metadata is loaded once and read-only thereafter; only score/award cvars are mutable, and they are managed directly as strings.
- **VM boundary efficiency:** All filesystem and cvar access goes through `trap_*` syscalls, allowing the UI to remain sandboxed; no direct file or cvar pointers leak into the VM.
- **Single-threaded, frame-driven:** No async loading, locking, or background tasks; all file I/O happens during `UI_InitGameinfo`, which runs before the first frame.

## Potential Issues

1. **No bounds checking on arena/bot array access:**
   - `UI_ParseInfos` allocates into `infos[count]` without checking if `count` exceeds `MAX_BOTS` / `MAX_ARENAS`. If parsing produces more entries than the array can hold, a buffer overflow is possible (though `max` parameter in the call should prevent this; still, defensive code would be safer).

2. **Linear search performance:**
   - `UI_GetArenaInfoByNumber` does a full O(n) scan every time it is called. For 256 arenas, this is acceptable, but a one-time index build would be better.

3. **Silent level discarding on misaligned tier counts:**
   - If a map pack has, e.g., 25 SP arenas and `ARENAS_PER_TIER=6`, the code silently discards the last arena. A warning is printed, but mappers may not notice and assume all levels were loaded.

4. **Cvar overwrite risk:**
   - `trap_Cvar_Set` calls in `UI_SetBestScore` and unlock commands directly modify cvar strings without transaction semantics. If multiple calls race (unlikely in single-threaded UI but worth noting), data could be corrupted.

5. **No validation of score/skill values in `UI_SetBestScore`:**
   - The function validates score (1–8) and skill (1–5) ranges, but does not validate the `level` parameter against `ui_numArenas`. A caller passing an invalid level will silently write to the wrong cvar entry.
