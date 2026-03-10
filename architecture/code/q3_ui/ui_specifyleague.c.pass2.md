# code/q3_ui/ui_specifyleague.c — Enhanced Analysis

## Architectural Role

This file implements a specialized Global Rankings (GRank) feature menu within the broader Q3A UI VM ecosystem. It bridges user interaction (player name entry, league selection) with engine/backend services through trap syscalls (`trap_CL_UI_RankGetLeauges`, cvar reads/writes). The menu slots into the standard UI menu stack (`UI_PushMenu`/`UI_PopMenu`) and demonstrates the classic sandbox pattern: UI VM code is completely isolated from game/server state and communicates exclusively via trapped syscalls and cvars.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/q3_ui/ui_main.c` or similar**: likely calls `UI_SpecifyLeagueMenu()` as a menu activation entry point from a parent rankings/account menu
- **Menu framework**: depends on `Menu_AddItem()`, `ScrollList_Key()`, `UI_PushMenu()`, `UI_PopMenu()` from `code/q3_ui/ui_qmenu.c` / `ui_atoms.c`
- **Event loop**: integrated into UI VM's per-frame input dispatch via registered `SpecifyLeague_Event` callbacks on each widget

### Outgoing (what this file depends on)
- **Engine trap syscalls** (from `code/client/cl_ui.c` dispatch):
  - `trap_CL_UI_RankGetLeauges()` — GRank extension syscall querying backend for leagues; not in vanilla Q3
  - `trap_Cvar_Set()`, `trap_Cvar_VariableStringBuffer()` — cvar read/write for league results and `sv_leagueName` output
  - `trap_R_RegisterShaderNoMip()` — asset pre-registration (renderer front-end)
- **Shared utilities** from `q_shared.c`: `Q_strncpyz()`, `Q_strncmp()`, `va()` string functions
- **Menu framework** from `ui_qmenu.c`/`ui_atoms.c`: standard widget infrastructure and menu stack management

## Design Patterns & Rationale

**Menu-widget framework (MVC-lite)**: All UI state lives in a single `specifyleague_t` struct; widgets reference it and invoke a central dispatcher (`SpecifyLeague_Event`). This avoids scattered callbacks and eases testing.

**Lazy backend queries**: The `playername` static cache prevents redundant backend calls; re-query only triggers on name-field focus loss with actual change. This is idiomatic for this era's resource-constrained UI loops.

**Fixed-size array caching**: `league_table[128]` and `leaguename_items[128]` use simple static arrays rather than dynamic allocation. The list widget holds pointers into `league_table[i].buff`—a classic pre-C99 pattern that avoids malloc but requires careful size bounds (`MAX_LISTBOXITEMS=128`, `MAX_LEAGUENAME=80`).

**Asset pre-caching**: All art (frames, buttons, logos) is registered synchronously in `SpecifyLeague_Cache()` before the menu displays. This ensures no stalls during interaction and is standard for this engine's rendering pipeline.

## Data Flow Through This File

1. **Init phase**: `UI_SpecifyLeagueMenu()` → `SpecifyLeague_MenuInit()` zeroes state, pre-caches art, pre-populates name field from `name` cvar, calls `SpecifyLeague_GetList()` for initial query.
2. **Query phase**: `SpecifyLeague_GetList()` copies field buffer to static `playername`, calls `trap_CL_UI_RankGetLeauges(playername)`, then reads backend results from numbered cvars (`leaguename1`, `leaguename2`, …) into `league_table[i].leaguename`. Pointer array `leaguename_items[i]` is pre-initialized to point into `league_table[i].buff` so the list widget automatically displays.
3. **User input phase**: `SpecifyLeague_Event()` handles scroll (up/down arrows), re-query on name change (focus loss + diff check), and back/confirm (writes selected league name to `sv_leagueName` cvar and pops menu).
4. **Exit phase**: Menu pops; caller handles `sv_leagueName` to activate the selected league server-side.

## Learning Notes

**GRank extension**: The `trap_CL_UI_RankGetLeauges` syscall is not part of vanilla id-tech-3; it's a Quake III Arena / early ioquake3 extension for online player rankings. The typo "Leauges" is preserved from original source (should be "Leagues"), indicating minimal post-release refactoring.

**Idiomatic patterns of early 2000s C**: Static globals, fixed-size arrays, explicit position/size initialization (virtual 640×480 space). Modern engines use dynamic layout, scene graphs, or data-driven UI definitions; this file shows the pre-scriptable-UI era.

**Sandbox model in action**: UI VM has zero direct access to `entityState_t`, `playerState_t`, or game state. All queries go through trap syscalls (`trap_CL_UI_RankGetLeauges`) or cvars (`leaguename1`, etc.). This enforces the VM→engine boundary and prevents UI exploits.

**Menu stack discipline**: Proper use of `UI_PushMenu()` / `UI_PopMenu()` allows nested menus without globals; callback-driven event routing keeps state mutations local to `SpecifyLeague_Event()`.

## Potential Issues

- **Unvalidated cvar reads** (lines ~105–108): `trap_Cvar_VariableStringBuffer()` results are not bounds-checked for malformed engine responses; a hostile engine could overflow `league_table[i].leaguename` (80-char buffer).
- **Dead bitmap**: `s_specifyleague.grletters` is initialized (line ~197) but never added via `Menu_AddItem()`. Likely incomplete feature or copy-paste from another menu.
- **Initialization order dependency**: File-static `league_table` and `leaguename_items` are uninitialized until `SpecifyLeague_MenuInit()` runs. If the list widget were accessed before init, it would read garbage pointers.
- **Static name cache**: `playername` is only updated in `SpecifyLeague_GetList()` and compared in `SpecifyLeague_Event()`. If the field is edited but never loses focus before menu exit, the change is silently discarded (though this is probably intentional—user can confirm with back button).
