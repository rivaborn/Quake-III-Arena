# code/q3_ui/ui_demo2.c — Enhanced Analysis

## Architectural Role
This file is a single **menu module** within the larger q3_ui VM subsystem, demonstrating how the legacy Q3A UI implements self-contained menu screens. It follows a standardized widget framework pattern (static menu state, callback-driven event dispatch, shader precaching) that repeats across all q3_ui menus (`ui_main.c`, `ui_servers2.c`, etc.). The demo menu sits at the intersection of three engine subsystems: the **renderer** (asset caching), **filesystem** (demo enumeration), and **console** (command execution), communicating exclusively through syscall wrappers (`trap_*`).

## Key Cross-References

### Incoming (who depends on this file)
- **UI dispatcher** (`ui_main.c`, `ui_atoms.c`): Calls `UI_DemosMenu()` when user selects "Demos" from main menu; this is the sole public entry point
- **Menu framework** (`ui_qmenu.c`): Provides `Menu_AddItem`, `Menu_DefaultKey`, `Menu_ItemAtCursor`, `ScrollList_Key` callbacks that drive the menu loop
- **Asset precacher**: `Demos_Cache()` may be called during level precache phase (alongside other UI modules' `*_Cache()` functions) to warm shader cache before menu draws

### Outgoing (what this file depends on)
- **Renderer syscalls**: `trap_R_RegisterShaderNoMip` — queues 9 UI artwork assets (back, go, arrows, frames) for GPU resident-set inclusion
- **Filesystem syscalls**: `trap_FS_GetFileList` — enumerates demos from `demos/` directory filtered by protocol-versioned extension (e.g., `dm_68`)
- **Cvar syscalls**: `trap_Cvar_VariableValue("protocol")` — reads protocol version to construct correct extension filter at init time
- **Console syscalls**: `trap_Cmd_ExecuteText(EXEC_APPEND, va("demo %s\n", ...))` — when user plays demo, executes console command in server frame
- **Menu framework**: `UI_ForceMenuOff`, `UI_PopMenu`, `UI_PushMenu` — stack operations on the global menu stack; `ScrollList_Key` — list navigation helper
- **String utilities** (qcommon): `va()`, `Com_sprintf()`, `strcpy()`, `strlen()`, `Q_stricmp()`, `Q_strupr()` — all compiled into Q3A's game library (`q_shared.c`)

## Design Patterns & Rationale

### Menu Framework Pattern
- **State container**: `demos_t s_demos` is a static module-local singleton holding all UI state (widgets, demo list, counts)
- **Widget composition**: Demonstrates the "menu = collection of typed controls" pattern: `menutext_s` (banner), `menubitmap_s` (artwork + buttons), `menulist_s` (scrollable list), all registered via `Menu_AddItem`
- **Callback dispatch**: All user interaction funnels through `Demos_MenuEvent`, which decodes widget ID and event type to branch behavior — standard Q3A UI convention
- **Syscall boundary**: Every interaction with engine (render, filesystem, console) goes through narrow `trap_*` ABI, enforcing sandbox isolation for QVM

### File Enumeration Pattern
Protocol-versioned demo enumeration is a recurring pattern across Q3A menus:
- Build extension filter string using **cvar-derived protocol version** (vs. hard-coded `dm_67` or `dm_68`)
- Call `trap_FS_GetFileList` with that filter to fetch only compatible demos
- This allows **single binary to run multiple protocol versions** without recompilation
- See also: `ui_servers2.c`, `cg_players.c` (model/skin enumeration)

### Precaching Strategy
`Demos_Cache()` is a **decoupled precacher** pattern — called during `Demos_MenuInit` but also (potentially) during level precache phases to ensure all assets are resident before menu draws. This matches patterns in `ui_main.c`, `cg_main.c` (cgame precache) and reflects Q3A's early 2000s GPU memory constraints (8–64 MB typical).

## Data Flow Through This File

```
User clicks "Demos" in main menu
  ↓
UI_DemosMenu() → Demos_MenuInit()
  ├─ trap_FS_GetFileList("demos", "dm_<protocol>", ...)  ← filesystem
  ├─ Parse returned name buffer, strip extensions, uppercase
  ├─ Demos_Cache() → trap_R_RegisterShaderNoMip ×9  ← renderer
  └─ Menu_AddItem(...) ×9 to register all widgets
  ↓
Menu framework loops, draws list, reads input
  ├─ User selects demo + presses "Go"
  ├─ Demos_MenuEvent(ID_GO, ...) fires
  ├─ UI_ForceMenuOff() clears menu stack
  ├─ trap_Cmd_ExecuteText(EXEC_APPEND, "demo <name>\n")  ← console
  └─ Engine's command parser executes demo playback
```

**Key state mutation:**
- `s_demos.names[NAMEBUFSIZE]`: Written by `trap_FS_GetFileList`, then mutated in-place (extension stripped, text uppercased)
- `s_demos.demolist[MAX_DEMOS]`: Pointer array built to slice into `names` buffer (zero-copy pattern)
- No persistent state across menu close/reopen; fresh enumeration on each `UI_DemosMenu()` call

## Learning Notes

### Q3A-Era UI Architecture
This file illustrates the **QVM sandbox + syscall boundary** design pattern that dominates Q3A's architecture:
- UI logic (menu system, widget state) lives in isolated QVM bytecode
- All engine interaction funnels through narrow, versioned syscall ABIs (`trap_R_*`, `trap_FS_*`, `trap_Cvar_*`, `trap_Cmd_*`)
- No direct memory access to engine data; all queries return copied or serialized results
- Enables **engine swapping** (renderer, filesystem, filesystem) without recompiling UI

### File Enumeration Pattern (Still Relevant)
The protocol-versioned file list pattern is idiomatic to Q3A's modular architecture and persists in modern Quake derivatives. Compare to modern engines:
- Modern engines often use **asset metadata registries** (database of available items) or **hardcoded lists**
- Q3A's approach is **lightweight** (minimal precache) and **data-driven** (filesystem controls what's available) but requires two-pass construction (enumerate, then parse in-place)

### Legacy UI Limitations
- **No runtime menu reloading**: Menu state is fully static after init; to change demo list, must reload UI module
- **Hard 128-demo cap** (`MAX_DEMOS`): Scales poorly for large demo archives
- **Simple string mutation**: In-place extension stripping and uppercasing of the file list buffer — modern engines would use temporary strings or structures
- **Callback-based input**: No per-widget update logic; all state changes are event-driven (common in 2000s UI systems, less so in modern reactive/data-driven UIs)

## Potential Issues

1. **Dead Code in `UI_DemosMenu_Key`** (line 121):
   - `item = Menu_ItemAtCursor(...)` is fetched but never used; result discarded before call to `Menu_DefaultKey`
   - No functional impact, but suggests copy-paste from other menu handlers or incomplete refactoring

2. **Hard-Coded Extension vs. Dynamic Protocol**:
   - File enumeration uses protocol-derived extension: `va("dm_%d", (int)trap_Cvar_VariableValue("protocol"))`
   - But extension **stripping** (line 262) hard-codes `.dm3`: `!Q_stricmp(demoname + len - 4, ".dm3")`
   - If protocol version used a different extension (e.g., `.dm_68`), extension stripping would fail silently, leaving garbage in display names
   - *Likely never manifested in production* because Q3 demo format was `.dm3` across all protocol versions, but indicates incomplete abstraction

3. **Array Bounds on `demolist`** (line 264):
   - Loop writes to `s_demos.list.itemnames[i]` (which points into `s_demos.demolist`)
   - `numitems` is clamped to `MAX_DEMOS` (128), so buffer overflow is prevented
   - However, `demolist` is declared as `char *demolist[MAX_DEMOS]` — static sized, and loop uses index `i` directly without bounds check
   - Risk is low because `numitems` is clamped, but a future refactor removing the clamp would cause out-of-bounds write

4. **No Error Recovery for Degenerate Empty-List Case**:
   - If no demos found, the "Go" button is hidden/disabled but menu remains open with placeholder "No Demos Found." string
   - User can navigate to it but cannot activate it — design is functional but UX is minimal (no confirmation dialog, no "back to main menu" hint)
