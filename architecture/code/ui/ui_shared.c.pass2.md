# code/ui/ui_shared.c — Enhanced Analysis

## Architectural Role

This file is the **core runtime engine of the MissionPack (Team Arena) data-driven UI subsystem**. It sits in the `ui` VM module, bridging between the engine (via `trap_*` syscalls) and the menu script data files. Unlike the legacy `q3_ui` module which implements menu structure in C code, this file provides a generalized widget framework where menus and items are parsed from script files at load time, making it highly customizable without recompilation. It is the sole implementation shared across both `ui` and `cgame` builds via conditional compilation flags.

## Key Cross-References

### Incoming (who depends on this file)
- **`code/ui/ui_main.c`** — VM entry point; calls `String_Init()` on `UI_INIT`, invokes `Menu_PaintAll()` on per-frame `UI_DRAW`, routes input via `Display_HandleKey()` and `Display_MouseMove()`
- **`code/ui/ui_atoms.c`** — Companion utility layer; parsing/display initialization
- **`code/ui/ui_syscalls.c`** — Syscall stub layer; resolves `trap_*` function pointers at runtime
- **`code/client/cl_ui.c`** — Client engine; hosts UI VM, calls `vmMain(UI_*)` to dispatch frame/input events
- **`code/cgame/cg_newdraw.c`** — cgame rendering; uses owner-draw IDs (`CG_SHOW_*` flags) and calls into `UI_*` owner-draw handlers for HUD composition

### Outgoing (what this file depends on)
- **Engine syscalls** via `displayContextDef_t *DC` vtable:
  - `trap_R_*` — all rendering (text, images, geometry)
  - `trap_S_*` — sound playback
  - `trap_Cvar_*` — cvar read/write
  - `trap_GetBindingBuf` / `trap_SetBinding` — key binding I/O
  - `trap_GetConfigString`, `trap_Cmd_ExecuteText` — engine state queries and command execution
- **`code/qcommon/q_shared.c`** — `COM_ParseExt`, `Q_stricmp`, `Q_strcat`, string utilities
- **`ui/menudef.h`** — shared constant vocabulary (widget types, feeder IDs, owner-draw IDs, visibility flags)
- **`code/renderer/tr_types.h`** — type definitions for owner-draw rendering contexts

## Design Patterns & Rationale

### Data-Driven Widget Framework
Menus are defined in `.menu` script files and parsed at runtime into `menuDef_t` and `itemDef_t` structures. This decouples UI layout/behavior from code, enabling:
- **Designer-friendly iteration**: Non-programmers can edit menu text, positioning, and behavior without recompilation
- **Dynamic customization**: Custom mods can define entirely new UI layouts without engine changes
- **Hot-reload capability**: Menus can be reloaded in theory, though the engine would need to explicitly free and reload

### Fixed-Size Memory Pools
Both string interning (`strPool[STRING_POOL_SIZE]` with `HASH_TABLE_SIZE=2048` buckets) and general allocation (`memoryPool` with bump allocator) use pre-allocated, fixed-size arenas. This is typical of late-1990s engine design (pre-generational GC era):
- **Predictable memory behavior** — no dynamic malloc fragmentation or GC pauses
- **Embedded-system friendliness** — suitable for consoles with limited/fragmented heaps
- **Deterministic initialization** — pool size known at compile time
- **Tradeoff**: exhaustion causes failure (string/alloc returns NULL) rather than graceful expansion

### Hash-Based Dispatch
Keyword parsing uses a two-level strategy:
1. **`hashForString()`** — compile-time string → hash → bucket lookup for fast O(1) command/keyword dispatch
2. **Chained collision handling** — `stringDef_t->next` chains and `keywordHash_t` hash-chain collision lists
- Enables rapid tokenization and per-command handler dispatch
- Interning avoids string duplication in memory

### Capture/Focus Model
**Mouse capture** (`itemCapture`, `captureFunc`, `captureData`), **key-bind capture** (`g_bindItem`, `g_waitingForKey`), and **text-field edit mode** (`g_editItem`, `g_editingField`) implement explicit state machines for input routing:
- Single-threaded design: only one item can have capture/focus at a time
- Simplifies event routing logic (no message queues or event bubbling)
- Well-suited to immediate-mode rendering paradigm where input is routed *during* paint

### Orbit and Transition Animations
Items support orbital motion (`item->window.offsetTime`, `item->window.effectTime`) and fade transitions (`item->window.fadeTime`), updated per-frame in `Item_Paint()` before rendering. This is a simple **temporal interpolation pattern** — advance animation state, then use it to modulate position/opacity during draw.

## Data Flow Through This File

**Initialization** (engine calls `vmMain(UI_INIT, ...)`):
1. `String_Init()` → reset all globals, build keyword hash tables
2. `Controls_GetConfig()` → read engine key-binding state into `g_bindings[]` array
3. Menu files parsed incrementally as engine calls `Menu_New(handle)` per script block

**Per-Frame Render** (engine calls `vmMain(UI_DRAW, ...)`):
1. `Menu_PaintAll()` → iterate all menus in `Menus[]` array
2. For each menu: `Menu_Paint()` → paints background/window, then per-item `Item_Paint()`
3. `Item_Paint()` → updates animation state (orbit, transition), then dispatches to type-specific paint (Text, ListBox, Slider, Model, Bind, etc.)
4. All rendering queued via `DC->drawText`, `DC->drawRect`, `DC->drawHandlePic`, etc., executed by engine render backend

**Input Dispatch** (engine calls `vmMain(UI_KEY, key, down, ...)`):
1. `Display_HandleKey()` routes to active menu stack top
2. `Menu_HandleKey()` checks:
   - **Bind-capture mode**: `Item_Bind_HandleKey()` → interpret as key binding, update `g_bindings[]`, emit `Controls_SetConfig()`
   - **Edit mode**: `Item_TextField_HandleKey()` → accumulate text input
   - **Normal mode**: item's key handler (listbox, slider, etc.) or fallback to navigation (PgUp/PgDn, arrow keys, Tab)
3. Matching items may execute attached scripts via `Item_RunScript()`

**Scripting** (user action or cvar change triggers command string):
1. `Item_RunScript(item, "open_menu main; set some_cvar 1;")` tokenizes by semicolon
2. Dispatches each token to `commandList` handlers (e.g., `Script_Open`, `Script_Set`) or falls back to `DC->runScript`
3. Side effects: menu transitions, cvar changes, sound playback, item visibility/color updates

## Learning Notes

### Idiomatic to Quake III / 2005 era:
- **Bump-pointer allocation** — no free list, only reset-on-reload
- **String interning** — manual deduplication via hash table + pool
- **Keyword dispatch** — hand-written hash tables + string matching (no reflection/introspection)
- **VM isolation** — UI code runs in separate QVM bytecode sandbox for security/portability
- **Per-type rendering** — no unified renderer abstraction; each widget type has its own paint function
- **Immediate-mode semantics** — input is dispatched *during* per-frame paint cycle, not queued
- **Single-threaded event loop** — no message pumps, observers, or async handlers

### Modern contrasts:
- **Dynamic allocation** — malloc/free or generational GC instead of fixed pools
- **Declarative UI** — YAML/XML/JSON instead of runtime parsing
- **Scripting languages** — Lua/Python instead of custom script tokens
- **Event queues** — asynchronous message dispatch instead of synchronous routing
- **Retained-mode or Immediate-mode GUI libraries** — Dear ImGui, Flutter, React instead of bespoke widgets
- **GPU-accelerated text** — signed-distance field fonts, glyph atlases instead of bitmap font rendering

### Connections to game engine concepts:
- **Data-driven architecture** — parallels entity-component-system pattern; behavior defined by data, not hardcoded type hierarchy
- **VM sandboxing** — similar to Lua/WASM security model; untrusted code in isolated runtime
- **Hierarchical state machines** — menu stack mirrors scene-graph or UI-tree depth stacking
- **Fixed resource pools** — classic resource management for embedded/console targets (trade flexibility for predictability)
- **Ownership model** — engine owns UI VM lifecycle; UI owns menus/items; items own child data — simple hierarchical ownership, no shared pointers

## Potential Issues

1. **String pool exhaustion**: `String_Alloc()` returns `NULL` if pool is full (line ~205), but callers are not visibly validated against NULL. Script definitions or interned strings that exceed `STRING_POOL_SIZE` will silently fail.

2. **Memory pool re-exhaustion across reloads**: `UI_InitMemory()` resets the bump pointer to zero on reload, but if the arena size is fixed and menus grow over time (via mods or updates), eventual OOM is guaranteed. No resizing or fallback mechanism.

3. **Unbounded script recursion**: `Item_RunScript()` can execute `Script_Open` which calls `Menu_New()`, triggering nested parsing and more scripts. No visible recursion depth check; deep nesting could overflow the stack.

4. **Global capture state coupling**: Multiple static captures (`itemCapture`, `captureFunc`, `g_bindItem`, `g_editItem`) assume single-threaded event dispatch. If the engine or cgame calls into UI asynchronously during menu handling, race conditions or inconsistent state could occur.

5. **Menu stack validity**: `menuStack[]` stores raw `menuDef_t*` pointers. If a menu is removed or overwritten in `Menus[]`, pointers in the stack become invalid, causing crashes on restore via `Menu_SetPrevCursorItem()` or `Menu_SetNextCursorItem()`.

6. **Focus restoration on item deletion**: Item-specific handlers (listbox, slider) maintain focus via `item` pointers; no validation that item still exists if menus are dynamically rebuilt. Stale pointers could cause segfaults.
