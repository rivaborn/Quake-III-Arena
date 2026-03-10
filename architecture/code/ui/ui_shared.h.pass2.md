# code/ui/ui_shared.h — Enhanced Analysis

## Architectural Role

This header defines the **complete UI abstraction layer** bridging the VM-based UI modules (`code/ui` and `code/q3_ui`) to the engine core. It serves as a **dependency-inversion boundary**: the `displayContextDef_t` vtable decouples UI rendering logic from renderer/sound/input implementation, enabling the same `.menu` scripts and `ui_shared.c` logic to run in the main menu VM and the cgame HUD VM without modification. The file is foundational to Quake III's **data-driven UI approach**, where menus are parsed from `.menu` script files and compiled into the type-safe C structures declared here.

## Key Cross-References

### Incoming (who depends on this file)
- **code/q3_ui/** and **code/ui/** — Both UI VMs include this header and implement the `vmMain` entry point that consumes the display context and menu API
- **code/cgame/** — The cgame HUD module includes this for owner-draw item IDs and the menu framework (cgame can render HUD menus identically to q3_ui)
- **code/client/cl_ui.c** — Routes all UI VM syscalls (indexed into `trap_*` functions); `Init_Display` is called here with the renderer/sound vtable pointers
- **code/renderer/** (indirectly) — Provides the `displayContextDef_t` vtable populated with renderer function pointers; called from `Init_Display`

### Outgoing (what this file depends on)
- **code/game/q_shared.h** — `vec4_t`, `qhandle_t`, `sfxHandle_t`, `fontInfo_t`, `glconfig_t`, `refEntity_t`, `refdef_t`, `pc_token_t`, `qboolean`
- **code/cgame/tr_types.h** — `refEntity_t`, `refdef_t` for 3D model preview support
- **../../ui/menudef.h** — Shared `ITEM_TYPE_*`, `FEEDER_*`, `CG_SHOW_*` constants (compile-time only, no linkage)
- **trap_PC_*** syscalls — Implemented in the VM syscall layer (`code/ui/ui_syscalls.c`, `code/cgame/cg_syscalls.c`); wraps the preprocessed script parser
- **trap_R_*** / **trap_S_*** / **trap_CL_*** — Renderer, sound, and client syscalls; vtable pointers populated by the engine in `Init_Display`

## Design Patterns & Rationale

**Vtable / Strategy Pattern**: `displayContextDef_t` is a classic vtable enabling **dependency inversion**. The UI code never calls the renderer directly; instead, it calls through function pointers populated at runtime. This allows the same UI code to run in both cgame (HUD) and main menu contexts with different backend implementations.

**Hierarchical Type System**: `windowDef_t` (base visual properties) → `itemDef_t` (adds scripting, type-specific data) → no further hierarchy, but type-specific substructures (`listBoxDef_t`, `editFieldDef_t`, `multiDef_t`, `modelDef_t`) are pointed to via `typeData` void pointer. This trades type safety for memory efficiency and flexibility (needed before C++ templating was practical).

**String Interning Pool**: All `const char*` fields point into a fixed `STRING_POOL_SIZE` buffer allocated at startup. This avoids heap fragmentation from thousands of small string allocations and enables O(1) string equality checks (pointer comparison). The pool is **never freed at runtime**; exhaustion is fatal (see `UI_OutOfMemory`).

**Parser Abstraction Layer**: Both text-stream (`Float_Parse`, etc.) and **PC token-stream** (`PC_Float_Parse`, etc.) variants are exposed. This allows `.menu` files to be preprocessed by a C preprocessor (`trap_PC_*` syscalls) before parsing, enabling `#include`, `#define`, conditional compilation — standard game-modding convenience.

**Feeder Pattern**: The `feederCount` / `feederItemText` / `feederItemImage` callbacks implement a **cursor/row-source abstraction**, allowing dynamic population of list boxes from arbitrary data sources (player list, server list, demo recordings, etc.) without coupling the UI to specific data structures.

**Cinematic Support in UI**: The `windowDef_t.cinematic` and `renderScene` vtable entry are unusual — they enable animated video backgrounds in menus (e.g., intro sequence). Reflects Quake III's "cinema-first" menu aesthetic design philosophy.

## Data Flow Through This File

**Script → Compiled Menu → Render → Input → Action**:
1. **Load phase**: `.menu` files are read, preprocessed via `trap_PC_*`, and parsed into `menuDef_t`/`itemDef_t` instances by `Menu_New` / `Menu_PostParse`. All strings are interned in the string pool.
2. **Render phase** (per-frame): `Menu_PaintAll` iterates all `MAX_OPEN_MENUS` open menus, calling `Menu_Paint` → `displayContextDef_t` function pointers (drawText, drawHandlePic, etc.). Each `itemDef_t` is rendered according to its type.
3. **Input phase**: `Display_HandleKey(key, down, x, y)` dispatches to `Menu_HandleKey` on the focused menu, which routes to the focused item's `action` script or special handlers (feeder selection, slider adjustment).
4. **Scripting**: Actions are executed via `runScript` vtable entry, which triggers server commands, cvar changes, or menu navigation.

**Special data paths**:
- **Feeder-driven list boxes**: `feederCount` determines list size; `feederItemText` / `feederItemImage` pull data per-frame, allowing real-time updates (e.g., server list refreshes).
- **Owner-draw items**: `ownerDrawItem` vtable entry allows cgame to inject custom rendering (e.g., weapon icons, HUD elements) identified by `ownerDraw` ID.

## Learning Notes

**Early Data-Driven UI** (pre-ECS era): This is a structured-data approach to UI that was innovative for 2005. Modern engines use scene graphs or ECS; this file shows an intermediate design: hierarchical structs with polymorphic subdata, no inheritance.

**Memory Efficiency**: The string pool and fixed-size menus/items arrays avoid dynamic allocation entirely. Useful for embedded/console platforms and predictable memory usage, but trades flexibility for constraints (`MAX_MENUS=64`, `MAX_MENUITEMS=96`).

**VM Boundary Design**: The `trap_PC_*` syscalls make the VM/engine boundary explicit. The UI VM cannot use the C preprocessor directly; the engine does preprocessing and returns tokens. This separation aids sandboxing and modding (`.menu` files are data, not code).

**Color Ranges & Themability**: `colorRangeDef_t` arrays in `itemDef_t` enable **conditional coloring** based on float ranges (e.g., status bar health: red when <25%, yellow 25–75%, green >75%). Advanced for a 2005 UI system.

**Feeder Abstraction**: Similar to database cursor/row-source patterns. Enables decoupling of list box rendering from data source, allowing menus to display dynamic content (server lists, chat logs, scoreboards) without hard-wiring UI structs to specific game data.

## Potential Issues

- **String Pool Exhaustion**: `STRING_POOL_SIZE` (384 KB) is fixed and never freed. Mods that load/unload many large menus could exhaust it; error recovery is minimal (`UI_OutOfMemory` check only at major allocation sites).
- **Type Erasure**: `itemDef_t.typeData` is `void*` — no runtime type information. Parsing code must track `type` separately and cast correctly; a mismatch causes memory corruption or segfault.
- **Hardcoded Limits**: `MAX_MENUS` (64), `MAX_MENUITEMS` (96), `MAX_OPEN_MENUS` (16), `MAX_LB_COLUMNS` (16) are compile-time constants. Large mod menus or nested menu structures can silently exceed these limits, causing data loss or undefined behavior.
- **No Validation Layer**: Parsed `.menu` files are not validated for semantic correctness (e.g., missing required fields, out-of-bounds cvar indices). Invalid input can crash the UI VM at runtime.
