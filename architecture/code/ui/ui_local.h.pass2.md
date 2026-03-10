# code/ui/ui_local.h — Enhanced Analysis

## Architectural Role

This file is the **central integration hub** of the Team Arena UI VM, serving three critical roles: (1) declaring the complete `trap_*` syscall ABI that forms the sandbox boundary between the UI and engine; (2) hosting all per-frame UI state (`uis`, `uiInfo`) as monolithic global singletons shared across every UI subsystem; and (3) providing both the legacy menu widget framework (for backwards compatibility with base-Q3A patterns) and type definitions for the new data-driven UI system (parsed from `.menu` files at runtime). Every `.c` file in `code/ui/` includes this header, making it the authoritative registry of the module's public interface and internal contract.

## Key Cross-References

### Incoming (who depends on this file)
- **All UI subsystem implementations** — every screen module (`ui_main.c`, `ui_atoms.c`, `ui_qmenu.c`, `ui_mfield.c`, `ui_players.c`, `ui_servers2.c`, etc.) includes this header to access type definitions, global state, and trap declarations.
- **Engine entry point** — `code/client/cl_ui.c` hosts the UI VM and dispatches syscalls through the `trap_*` table defined here; the engine calls `vmMain(UI_INIT, ...)`, `vmMain(UI_REFRESH, ...)`, `vmMain(UI_KEYEVENT, ...)` — these are routed to `UI_Init()`, `UI_Refresh()`, `UI_KeyEvent()` respectively (see ui_main.c).
- **cgame VM** — defines owner-draw IDs and `CG_SHOW_*` flags (referenced in `ui_public.h` includes) consumed by `code/cgame/cg_newdraw.c` for HUD visibility coordination.

### Outgoing (what this file depends on)
- **`../game/q_shared.h`** — foundational types (`vec3_t`, `vec4_t`, `qboolean`, `sfxHandle_t`, `qhandle_t`, `vmCvar_t`, keynum_t`).
- **`../cgame/tr_types.h`** — renderer types (`refEntity_t`, `refdef_t`, `glconfig_t`, `trRefEntity_s`) needed by `playerInfo_t` (player model preview), and `lerpFrame_t` (skeletal animation state).
- **`ui_public.h`** — the exported VM entry enum (`uiExport_t`, `uiMenuCommand_t`, `uiClientState_t`) defining the engine↔UI ABI contract.
- **`keycodes.h`** — `K_*` constants used by menu input handling; no runtime dependency, purely for symbolic key codes.
- **`../game/bg_public.h`** — `weapon_t`, `animation_t`, game type enums (`GT_FFA`, `GT_TOURNAMENT`, etc.) used by `uiInfo_t` (map/tier lists).
- **`ui_shared.h`** — data structures for the script-parsed menu system (`displayContextDef_t` and related types).

## Design Patterns & Rationale

### Legacy Menu Widget System (MTYPE_*)
The `menuframework_s` / `menucommon_s` hierarchy is a **type-dispatch pattern** using a base struct and embedded specializations (`menufield_s`, `menuslider_s`, `menulist_s`, etc.). Each widget stores a `type` field and a `void *parent` pointer to its container. The framework drives widgets via `Menu_DefaultKey()` / `Menu_Draw()` callbacks that manually dispatch on `generic.type`. This reflects late-2000s C game architecture before widespread use of inheritance/polymorphism.

**Rationale:** Pre-compiled menu screens (in C) needed a lightweight widget system that compiled to QVM bytecode. Avoiding virtual function pointers (which would require indirection overhead in the VM) led to the manual dispatch pattern.

### Monolithic State Aggregates (uis, uiInfo)
Rather than module-local state, **all UI runtime data is centralized** in two globals:
- `uiStatic_t uis` — transient per-frame state (time, mouse, GL config, common asset handles).
- `uiInfo_t uiInfo` — persistent session state (server lists, map/tier lists, player models, scores, mod list).

**Rationale:** Game engines of this era optimized for cache locality and deterministic initialization. A monolithic struct makes frame setup/teardown predictable and allows zero-allocation rendering loops. Modern UIs have moved to reactive/ECS patterns, but this was the era-appropriate design.

### Trap Syscall Abstraction
The `trap_*` function declarations form a **complete abstraction boundary**: the UI VM never calls engine symbols directly; all I/O, rendering, sound, filesystem, and networking goes through this indexed ABI. The engine implements the dispatch table in `code/client/cl_ui.c:CL_UISystemCalls()`.

**Rationale:** VM sandbox enforcement. A malicious or buggy UI DLL cannot crash the engine or access forbidden memory. The syscall index maps 1:1 to a `qvm_t::systemCalls[]` array.

### Coexistence of Legacy and New UI Systems
The header declares **both** the old C-based menu widget framework (`Menu_*`, `MenuField_*`, scroll lists) and types for a script-parsed menu system (via `displayContextDef_t` in `ui_shared.h`). The implementation uses both interchangeably for different screens.

**Rationale:** Team Arena was a content-driven expansion pack; the new data-driven UI system allowed artists/designers to create menus in a text-based format without recompiling. Legacy screens (options, ingame menu) kept the old C widget system for stability.

## Data Flow Through This File

1. **Initialization** — `UI_Init()` (ui_main.c) is called once at module load:
   - Calls `Menu_Cache()` to precache asset handles into `uis`.
   - Loads cvar defaults (`UI_RegisterCvars()`), maps, bots, arenas into `uiInfo`.
   - Calls `UI_LoadMenus()` to parse `.menu` script files.

2. **Per-Frame Update** — `UI_Refresh()` is called from `code/client/cl_main.c:CL_Frame()`:
   - Checks `uis.time` (from engine via trap) to drive animations.
   - Dispatches to the active menu's `draw()` callback.
   - Each menu renders 2D UI into the 640×480 virtual space via `trap_R_DrawStretchPic()`, `trap_R_DrawChar()`, etc.

3. **Input Dispatch** — `UI_KeyEvent()` and `UI_MouseEvent()` (from `code/client/cl_input.c`):
   - Route to the topmost menu on the stack (via `uis.activemenu`).
   - Call that menu's `key()` callback with the key code or mouse position.
   - Callbacks update `uiInfo` state (e.g., server browser filters, selected player) or invoke actions (connect, start game).

4. **Resource Binding** — Server/player/map lists are built into `uiInfo` during init and refreshed on demand:
   - `UI_ArenaServersMenu()` → `ArenaServers_Cache()` → populates `uiInfo.serverList[]`.
   - `UI_PlayerModelMenu()` → `PlayerModel_Cache()` → loads character images/models into `uiInfo.playerList[]`.

5. **Output** — All rendering and sound through trap syscalls:
   - `trap_R_*()` → engine's `re.Draw*()` callbacks → renderer (code/renderer/).
   - `trap_S_StartSound()` → engine's sound mixer (code/client/snd_*.c).
   - `trap_Cvar_Set()` / `trap_Cvar_VariableString()` → engine console var system.

## Learning Notes

### Idiomatic to This Era
- **Monolithic state aggregates** — modern UIs (React, Unreal UMG, Unity UI) are compositional or reactive; this design trades flexibility for predictability and low-memory overhead.
- **Immediate-mode drawing** — each frame, menus redraw from scratch (no retained scene graph). Modern engines retain a scene tree and only update changed subtrees.
- **Manual widget dispatch** — instead of virtual methods, a central `Menu_DefaultKey()` switch on `type`. This avoided VM indirection overhead.
- **Dual path (legacy + data-driven)** — coexistence shows the transition from artist-unfriendly compiled C code to designer-friendly scripting. Modern engines unified on scripting from the start.

### Connection to Broader Architecture
- **VM Sandbox** — `trap_*` syscalls enforce that the UI cannot access filesystem, network, or engine memory directly. The engine validates all calls before execution.
- **Syscall ABI versioning** — the `trap_*` declarations form a versioned contract. Engine and VM must agree on opcode indices; a mismatch causes syscall errors.
- **Renderer coupling** — UI draws into the same 2D layer as cgame HUD (code/cgame/cg_draw.c); both VMs share color constants (`menu_text_color`, etc.) defined here.
- **Server browser integration** — `serverStatus_t` and related types show the UI needs real-time network feedback (ping lists, MOTD); the server query mechanism is driven from the UI VM but dispatched to the engine.

### Modern Contrast
- **ECS UIs** — Unreal/Unity/Godot now use entity-component systems for UI, where each button/panel is an entity with updatable properties.
- **Reactive frameworks** — React, Vue, SwiftUI automatically diff and patch the UI; no manual `draw()` callback each frame.
- **Asset streaming** — modern engines load assets on-demand with ref-counting; this code preloads everything into `uis` and `uiInfo` at startup.
- **Data binding** — new frameworks bind cvar/state changes to UI automatically; here, cvars are manually synced in `UI_UpdateCvars()`.

## Potential Issues

### Not Easily Fixable Without Refactor
1. **Monolithic `uiInfo_t`** (est. ~200+ KB based on arrays and strings) — holds all UI state in one struct. Cache line pollution for unrelated subsystems (e.g., player preview data packed next to server list data). Modern approach: separate concerns into module-local state.

2. **No modularization of screen state** — each new UI screen (e.g., new game mode menu) adds fields to `uiInfo_t`. After ~10 menu systems, the struct becomes unmaintainable. Modern approach: per-screen state objects.

3. **Legacy widget system inflexibility** — adding a new widget type (`MTYPE_NEWTYPE`) requires modifying `Menu_DefaultKey()`, `Menu_Draw()`, and every affected menu. The new script-based system is more flexible but only used selectively.

4. **Network coupling** — `serverStatusInfo_t` and `pendingServerStatus_t` (line 920+) mix UI state with network plumbing. Server queries should ideally be decoupled from UI refresh.

### Observable in This File
5. **Duplication** — `ui_server1` through `ui_server16` (lines 73–88) are manually unrolled instead of using an array. This suggests the cvar system doesn't support dynamic registration.

6. **Global color arrays** — lines 294–311 declare ~15 `vec4_t` color constants. Centralizing color management here is good, but modern UIs theme colors dynamically.

---

## Summary

This header is **the central nervous system** of the Team Arena UI VM. It declares a complete sandbox ABI, hosts monolithic state aggregates shared across all screens, and bridges two eras of UI design (legacy C-based widgets and new data-driven scripts). The architecture reflects late-2000s game engine best practices: predictable per-frame refresh, minimal dynamic allocation, and tight VM sandboxing. Modern engines have moved to reactive, compositional, and data-bound UI frameworks—but for its time, this design was sound and maintainable.
