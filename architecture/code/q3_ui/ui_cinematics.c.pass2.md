# code/q3_ui/ui_cinematics.c — Enhanced Analysis

## Architectural Role

This file implements a single UI menu within the legacy q3_ui VM module (see: Architecture Overview, UI VMs section). It participates in the menu stack system (`UI_PushMenu`/`UI_PopMenu`) managed by `ui_atoms.c`, handling cinematic selection and playback initialization. The cinematics menu acts as a bridge between player intent (button selection) and engine execution: converting menu selections into `disconnect; cinematic <name>.RoQ` commands that pause gameplay and invoke the RoQ video player (hosted in `code/client/cl_cin.c`). It also gates access to tier completions via `UI_CanShowTierVideo`, enforcing single-player progression before allowing cutscene replay.

## Key Cross-References

### Incoming (Callers)
- **`ui_atoms.c`**: Owns the menu command dispatcher that invokes `UI_CinematicsMenu()` and `UI_CinematicsMenu_f()` when the cinematics menu is activated from the main menu or console
- **`ui_main.c`**: Likely registers `UI_CinematicsMenu_f` as a console command via `Cmd_AddCommand` during UI init
- **Menu framework**: `ui_qmenu.c` calls the registered callbacks (`UI_CinematicsMenu_Event`, `UI_CinematicsMenu_BackEvent`) when the player presses a key on an item

### Outgoing (Dependencies)
- **`ui_atoms.c`**: `UI_PopMenu()`, `UI_PushMenu()` manage the menu stack lifecycle
- **`ui_qmenu.c`**: `Menu_AddItem()` registers each widget with the menu framework; `Menu_SetCursorToItem()` repositions cursor in `UI_CinematicsMenu_f`
- **`ui_gameinfo.c`**: `UI_CanShowTierVideo(tier_num)` checks unlock status for each tier; gates via `QMF_GRAYED` flag
- **Engine syscalls** (`trap_*` in `ui_syscalls.c`):
  - `trap_Cvar_Set( "nextmap", ... )` — stores the selected cinematic index for post-disconnect handoff
  - `trap_Cmd_ExecuteText( EXEC_APPEND, ... )` — queues the disconnect + cinematic command for immediate execution
  - `trap_R_RegisterShaderNoMip()` — precaches UI art textures (back buttons, frame borders)

## Design Patterns & Rationale

**Stateful persistent menu struct**: `cinematicsMenuInfo` is a static global struct containing all 14 menu items (banner, frames, 10 cinematics, back button). This pattern avoids malloc/free in the hot-path and simplifies callback data access via pointer-to-menu-item.

**Event-driven callback system**: Each button stores a function pointer (`callback`) invoked with the menu item and event type. This is idiomatic to id Tech 3–era UI frameworks and predates modern signal/slot or observer patterns.

**Index-based lookup via ID offset**: Cinematics are indexed as `ID_CIN_IDLOGO` (11) through `ID_CIN_END` (20). The callback computes `n = id - ID_CIN_IDLOGO` to index `cinematics[]` array. This tight coupling avoids a separate enum-to-string mapping but makes reordering fragile.

**Conditional gating via grayed items**: Rather than hiding cinematics, the menu shows all 10 buttons but sets `QMF_GRAYED` to disable interaction. This keeps menu layout stable and communicates "locked" state clearly to the player.

**Demo version branching**: The special case for `demoEnd.RoQ` (line 108) is explicit, not abstracted. This reflects Q3A's product strategy: the demo had a shorter ending cinematic.

## Data Flow Through This File

1. **Initialization phase** (when cinematics menu is opened):
   - `UI_CinematicsMenu()` → `UI_CinematicsMenu_Init()` → zeroes state, constructs 14 widgets, calls `UI_CanShowTierVideo()` for each tier to conditionally gray, calls `Menu_AddItem()` for each widget
   - `UI_CinematicsMenu_Cache()` precaches shader art (invoked from `Init`)
   - `UI_PushMenu()` adds the menu to the stack; menu framework now owns input dispatch

2. **Interaction phase** (per-frame during menu display):
   - Menu framework calls `UI_CinematicsMenu_Event()` when player presses Enter on a cinematic button
   - Callback computes cinematic index, sets `nextmap` cvar with the index (for server-side tracking), issues a disconnect + cinematic command

3. **Playback phase** (handled externally):
   - Engine processes the queued command: `disconnect` drops the current connection, `cinematic <name>.RoQ` triggers `code/client/cl_cin.c` to load and play the video

## Learning Notes

**Snapshot of early-2000s menu architecture**: This file exemplifies the struct-of-menu-items pattern used before more declarative (XML/script-based) UI toolkits. Compare to `code/ui` (MissionPack), which moves to script-parsed menus—an architectural evolution visible within the same codebase.

**QVM/DLL sandbox enforcement**: All engine interaction flows through indexed `trap_*` syscalls, not direct linking. The `cinematicsMenuInfo` global is QVM-local; the engine never accesses it. This clean boundary enabled Q3A's QVM/DLL/native-DLL flexibility.

**Tier unlock integration**: The call to `UI_CanShowTierVideo()` shows how the UI VM consults game state (completion progress) held server-side via the `uis` global cvar cache. This is a simple form of data binding: UI reads the underlying cvar state.

**Cursor positioning offset (`+3`)**: The `items[n + 3]` in `UI_CinematicsMenu_f` skips the banner (0), framel (1), and framer (2) to reach the first selectable cinematic item (3). This reveals that `Menu_AddItem()` appends to a linear array, and the UI code must know the internal ordering to address items correctly—a brittleness modern menu systems avoid via named or unique-ID indexing.

## Potential Issues

- **Out-of-bounds in `UI_CinematicsMenu_f`**: `atoi( UI_Argv(1) )` is unchecked; a negative or large index passed to `items[n + 3]` could crash. (Likely protected by console command validation elsewhere, but not locally.)
- **Tier 8 vs. 7 ambiguity**: `UI_CanShowTierVideo(8)` for the END cinematic but only tiers 1–7 are listed. Either END is conceptually "tier 8" in progression logic (in `ui_gameinfo.c`), or there's off-by-one confusion. Not inferable from this file alone.
- **`cinematics[]` alignment risk**: If someone reorders the `ID_CIN_*` defines but forgets to reorder the array, indices will silently mismatch. A static assertion or name-based lookup would be safer.
