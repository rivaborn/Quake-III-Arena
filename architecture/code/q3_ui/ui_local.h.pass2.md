# code/q3_ui/ui_local.h — Enhanced Analysis

## Architectural Role

This file defines the complete internal contract for the legacy **base-Q3A UI VM**—one of two competing UI implementations in the engine (alongside MissionPack's data-driven `code/ui`). Unlike the newer ui module (which uses script-parsed `.menu` files), q3_ui is a hand-coded C menu framework using a C-style "vtable-lite" polymorphism pattern. It serves as the central namespace through which all screen implementations (`ui_main.c`, `ui_servers2.c`, etc.) coordinate menu state, asset loading, and syscall dispatch to the engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Engine `CL_UISystemCalls` dispatcher** (`code/client/cl_cgame.c`): Invokes q3_ui VM entry points (`vmMain` → `UI_Init`, `UI_Refresh`, `UI_KeyEvent`, `UI_MouseEvent`, `UI_Shutdown`) for every UI interaction in the client loop
- **All q3_ui screen modules** (`ui_menu.c`, `ui_servers2.c`, `ui_players.c`, etc., ~40 files): Each depends on this header to access the menu framework (`Menu_*` functions), render globals (color constants, sound handles), player preview infrastructure (`lerpFrame_t`, `playerInfo_t`), and VM lifecycle entry points
- **Audio subsystem** (`snd_dma.c`): Uses registered sound handles (`menu_in_sound`, `menu_move_sound`, etc.) defined here
- **Renderer** (`tr_main.c` via `trap_R_*` syscalls): Receives refdef submissions from `UI_Refresh` → `Menu_Draw` → per-screen draw callbacks

### Outgoing (what this file depends on)
- **`ui_public.h` (new UI ABI)**: Provides `uiExport_t`, `uiImport_t`, `uiMenuCommand_t` enums; q3_ui hijacks the new header but downgrades `UI_API_VERSION` to 4 for backward compatibility
- **`game/q_shared.h`**: Base types (`vec4_t`, `sfxHandle_t`, `vmCvar_t`, `qboolean`), string utilities, math macros
- **`cgame/tr_types.h`**: `refEntity_t`, `refdef_t`, `glconfig_t`, `polyVert_t` for 3D player rendering
- **`game/bg_public.h`**: `weapon_t`, `animation_t`, `MAX_ANIMATIONS`, game-type enums for player model setup
- **`keycodes.h`** (local): Key constants (`K_CHAR_FLAG`, etc.) for input dispatch
- **`qcommon` engine**: Via declared `trap_*` syscall wrappers (200+ functions crossing VM boundary)
- **Per-screen implementations**: Forward-declares all `UI_*Menu()`, `*_Cache()` functions they define

## Design Patterns & Rationale

**C-style Polymorphism (Pre-OOP)**: Menu widgets use `menucommon_s` as a base "vtable-like" struct embedded in derived types (`menufield_s`, `menuslider_s`, etc.). Callbacks (`draw`, `key`, `callback`, `ownerdraw`) are function pointers populated per-widget. This avoids C++ overhead and is typical of mid-2000s game engines targeting QVM (bytecode) where code size matters.

**Dual VM/DLL Execution Model**: The `trap_*` declarations represent a version-agnostic ABI—all function signatures are stable across VM and native DLL builds. Version pinning (`UI_API_VERSION = 4`) ensures the engine can reject mismatched modules at load-time.

**Global Resource Pooling**: Shared colors (`menu_text_color`, `color_red`, etc.) and sounds (`menu_in_sound`, `weaponChangeSound`) are declared as extern globals, populated once at `Menu_Cache()` startup, and reused across all screens. This avoids repeated texture/shader lookups and sound registrations.

**Menu Stack Architecture**: The `menuframework_s` draw/key callback pattern supports a LIFO stack (`uis.menuStack` per first-pass). Only the topmost menu draws and receives input. This design predates modern scene graphs and UI frameworks but is simple and effective.

**Sentinel Polymorphism**: The `MTYPE_*` constants (0–10) and corresponding struct types encode widget kind in the `menucommon_s.type` field. Dispatcher code (e.g., `Menu_DefaultKey`) switches on type and casts `generic` to the appropriate derived type. This is fragile by modern standards but avoids runtime type identification overhead.

## Data Flow Through This File

1. **Initialization** (`UI_Init` → `UI_RegisterCvars` → `Menu_Cache`): 
   - Loads all shader/sound assets via `trap_*` syscalls
   - Populates global color/sound handles
   - Calls per-screen `*_Cache()` functions to initialize screen-specific assets

2. **Per-Frame Update** (`UI_Refresh(realtime)`):
   - Increments timing state in `uis`
   - Calls topmost menu's `menuframework_s.draw()` callback
   - That calls per-widget draw handlers in sequence via `Menu_Draw()`
   - Each widget calls `trap_R_DrawStretchPic()`, `trap_SCR_DrawString()`, or custom `ownerdraw` callback

3. **Input Dispatch** (`UI_KeyEvent(key)` → `Menu_DefaultKey()`):
   - Routes to topmost menu's `key` handler
   - That iterates items, advancing cursor or calling per-widget `menucommon_s.key()` callback
   - Callbacks can activate items, change values, or push new menus via `UI_PushMenu()`

4. **Player Model Rendering** (in `ui_players.c`):
   - `UI_PlayerInfo_SetModel()` populates a `playerInfo_t` with skeletal animation state
   - `UI_DrawPlayer()` submits refEntity for legs, torso, head, weapon to renderer via `trap_R_AddRefEntityToScene()`
   - Uses `lerpFrame_t` to interpolate animation frames smoothly

## Learning Notes

**Idiomatic Patterns of Q3A Era**: 
- **Function-pointer callbacks over virtual methods**: Avoids C++ vtable indirection; all dispatch is explicit and inlinable
- **Global singleton state (`uis`)**: Central timing and menu stack; simpler than object-oriented scene graphs but harder to multi-thread (and Quake 3 doesn't attempt to)
- **Compile-time polymorphism (MTYPE_* enums) over runtime type info**: No RTTI; all type knowledge is baked at compile time

**Contrast to Modern UI Engines**:
- Modern frameworks (ImGui, Qt, React) use hierarchical component trees; q3_ui uses flat item arrays in a single menu struct
- Modern engines decouple rendering from layout; q3_ui bundles x/y position directly in `menucommon_s`
- Modern systems support themes/styling at runtime; q3_ui hardcodes colors at init time

**Snapshot of VM/Engine Boundary Design**: The 200+ `trap_*` function signatures show how a VM-hosted subsystem stays insulated from engine internals. Every syscall crossing is explicit, versioned, and type-checked at load time. This pattern also applies to cgame and game VMs.

## Potential Issues

1. **Version Pinning Fragility**: Overriding `UI_API_VERSION` to 4 after including `ui_public.h` (which may assume version ≥ 5) risks silent ABI mismatch if the header's struct layouts change. No runtime assertion validates this.

2. **Callback Dispatch Brittleness**: The `Menu_DefaultKey()` dispatcher is switch-based on `MTYPE_*` constants. Adding a new widget type requires modifying a central dispatcher function—poor modularity by modern standards.

3. **Color Constant Duplication**: Multiple color constants (e.g., `color_red` vs `menu_red_color` vs `text_color_*`) may lead to inconsistent theming. No centralized theme definition or validation.

4. **Cvar Sync Latency**: The 40+ extern `vmCvar_t` declarations must be synchronized via `trap_Cvar_Get()` in `UI_UpdateCvars()`. If a cvar changes in-engine mid-frame, the UI won't see it until next frame—potential desync in competitive settings.

5. **Player Preview Asset Leak Risk**: `UI_RegisterClientModelname()` is called per player model change, but if `Hunk_Alloc()` is called without freeing prior allocations, the hunk can fragment. No explicit free-on-exit for model assets.

---
