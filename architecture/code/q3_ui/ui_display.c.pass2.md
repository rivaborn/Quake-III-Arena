# code/q3_ui/ui_display.c — Enhanced Analysis

## Architectural Role
This file is a UI **subsystem bridge** that exposes two hardware-facing cvars (brightness/gamma and screen viewport size) to the player within the menu stack. It participates in the Q3A legacy menu module (`q3_ui`), which runs as an isolated QVM and communicates with the engine exclusively through VM syscall wrappers (`trap_*`). Display options affect two independent subsystems: the **Renderer** (`r_gamma`) and **cgame** (`cg_viewsize`), making this a lightweight integration point between UI, rendering, and client-game logic.

## Key Cross-References

### Incoming (who depends on this file)
- **UI main module** (`code/q3_ui/ui_main.c` / `ui_atoms.c`): Calls `UI_DisplayOptionsMenu()` when the player selects "Display" from the system setup menu
- **Menu framework** (`code/q3_ui/ui_qmenu.c`): Drives per-frame input dispatch to the topmost menu (`displayOptionsInfo.menu`) via generic `Menu_Draw()` and `Menu_DefaultKey()` loops
- **Sibling option screens** (`ui_video.c`, `ui_sound.c`, `ui_network.c` via implied entry points): Exist at the same menu level; navigation to/from them via `UI_PopMenu()` + push

### Outgoing (what this file depends on)
- **Menu framework** (`ui_qmenu.c`, `ui_atoms.c`): `Menu_AddItem()`, `Menu_SetCursorToItem()`, `UI_PushMenu()`, `UI_PopMenu()` for stack and widget management
- **Renderer syscalls** (`trap_R_RegisterShaderNoMip`): Pre-cache art assets (frame borders, back button)
- **Cvar syscalls** (`trap_Cvar_SetValue`, `trap_Cvar_VariableValue`): Read/write `r_gamma` and `cg_viewsize`
- **Global UI state** (`uis`): Query `glconfig.deviceSupportsGamma` to conditionally gray out brightness control
- **Sibling menu entry points** (`UI_GraphicsOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`): Navigate to adjacent option screens
- **Shared constants**: `color_red`, `color_white`, `PROP_HEIGHT`, `BIGCHAR_HEIGHT`, `MTYPE_*`, `QMF_*` flags

## Design Patterns & Rationale

### Pattern: Menu Stack Navigation
The display options file participates in a **push-pop stack pattern** for modal dialogs. When the user selects this screen, `UI_DisplayOptionsMenu()` initializes the menu and pushes it. When the user clicks back or navigates to a sibling screen, `UI_PopMenu()` dismisses it. This is idiomatic to legacy game UIs (Quake III, Half-Life) and avoids per-frame re-initialization.

**Why this way**: Simplifies input dispatch (top-of-stack menu gets all events), supports arbitrary nesting depth (e.g., sub-menus within option screens), and keeps per-menu state isolated.

### Pattern: Flat Struct Aggregation
All widgets for this screen are bundled into one static `displayOptionsInfo_t` struct. No dynamic allocation, no widget lists—compile-time-known layout.

**Why this way**: Maximizes performance and simplicity; all widgets exist for the lifetime of the module. The UI is largely static at runtime; dynamic behavior (focus pulsing, value changes) is handled by the menu framework, not per-widget allocation.

### Pattern: Cvar-Based State Synchronization
The UI doesn't maintain its own shadow copy of `r_gamma` or `cg_viewsize`. Instead, it:
- **On init**: Seeds slider positions from current cvar values (`trap_Cvar_VariableValue`)
- **On activation**: Pushes new values back to cvars (`trap_Cvar_SetValue`)

The renderer and cgame then read these cvars at their own pace.

**Why this way**: Decouples the UI from the renderer and cgame. The UI module doesn't need to know where those subsystems live or how they parse brightness; it just writes the cvar and the appropriate subsystem reacts. This is Q3A's primary IPC mechanism for cross-module settings.

### Pattern: Hardware Capability Gating
The brightness slider is grayed out (`QMF_GRAYED` flag) if `!uis.glconfig.deviceSupportsGamma`. The renderer populates `glconfig` at startup after querying driver capabilities.

**Why this way**: Gracefully degrades when the GPU doesn't support gamma correction. Instead of crashing or ignoring input, the UI informs the player the feature is unavailable.

## Data Flow Through This File

1. **Initialization** (`UI_DisplayOptionsMenu`):
   - Calls `UI_DisplayOptionsMenu_Init()` which zeroes `displayOptionsInfo` and positions every widget
   - Queries renderer state (`uis.glconfig.deviceSupportsGamma`) to conditionally disable brightness control
   - Queries current cvar values to seed slider positions
   - Pushes menu onto the UI stack and sets cursor to the DISPLAY tab to mark it as active

2. **Per-Frame Rendering** (driven by `ui_qmenu.c`):
   - Generic `Menu_Draw()` iterates over all widgets in `displayOptionsInfo.menu` and renders them
   - Sliders and text buttons are drawn with focus pulsing or highlight depending on selection state

3. **Input Dispatch** (driven by `ui_qmenu.c`):
   - User presses up/down/left/right to navigate widgets
   - User presses ENTER to activate focused widget
   - All events route through `UI_DisplayOptionsMenu_Event()` callback (unified dispatcher)

4. **Action** (on `QM_ACTIVATED`):
   - **ID_BRIGHTNESS**: `trap_Cvar_SetValue("r_gamma", value / 10.0f)` → renderer applies next frame
   - **ID_SCREENSIZE**: `trap_Cvar_SetValue("cg_viewsize", value * 10)` → cgame applies next frame
   - **ID_GRAPHICS/SOUND/NETWORK**: `UI_PopMenu()` + push sibling screen
   - **ID_BACK**: `UI_PopMenu()` back to parent menu

5. **Teardown** (implicit on `UI_PopMenu`):
   - Menu is removed from stack; `displayOptionsInfo` state persists in memory but becomes inactive
   - Next time the menu is opened, `UI_DisplayOptionsMenu_Init()` zeros and rebuilds it

## Learning Notes

### Idiomatic Q3A Patterns
- **Unified event dispatch via ID**: All interactive widgets emit events with a numeric ID, routed through a single switch statement. Modern frameworks use callbacks per-widget, but Q3A's monolithic dispatcher is simpler and avoids per-widget function pointers.
- **Static menu initialization**: Menu items are not created dynamically; they're all declared and positioned upfront. This trades flexibility for startup speed and predictability.
- **Cvar-as-IPC**: Rather than calling renderer/cgame functions directly, the UI writes cvars and lets those subsystems poll. This enforces loose coupling at the cost of eventual consistency (a frame delay before changes apply).

### Contrast with Modern Engines
- **Modern UI frameworks** (Dear ImGui, Unreal UMG) use immediate-mode or reactive paradigms; Q3A uses retained-mode with manual push/pop management.
- **Modern ECS engines** might decompose this into a Brightness subsystem, a ScreenSize subsystem, etc., each observing cvar changes. Q3A bundles it all in one menu file.
- **Modern engines** often use hot-reload and data-driven menu definitions (XML, JSON, visual editor). Q3A hard-codes widget layout in C.

### Connection to Engine Concepts
- **Subsystem Boundaries**: This file exemplifies how Q3A enforces module isolation. The UI VM cannot directly call renderer functions; it must use `trap_*` syscalls and cvars. This is enforced by the VM sandbox (`dataMask` in `code/qcommon/vm.c`).
- **Configuration as Data**: CVars are Q3A's foundational config system. Every engine parameter (resolution, gamma, sound volume, network rate) is a cvar, persistent across sessions via `autoexec.cfg`, and queryable by any module.
- **Two-Phase Menu Rendering**: The menu framework calls `trap_R_AddDirtyRect()` or equivalent to mark regions needing redraw, then `trap_R_RenderScene()` to flush. This file doesn't directly manage rendering; it just populates widget state and lets the framework drive the renderer.

## Potential Issues

1. **No Bounds Validation on Cvar Write**: When the user adjusts the brightness slider, the code writes `curvalue / 10.0f` directly to `r_gamma` without checking if that's a valid range for the renderer. If the cvar system or renderer doesn't validate, out-of-range gamma could cause visual corruption or GPU stalls. (Not necessarily a bug in this file, but a systemic risk if the renderer doesn't clamp.)

2. **Slider Range Mismatch**: Brightness range is [5, 20] (maps to r_gamma [0.5, 2.0]). If a buggy save/load cycle or cvar console command sets `r_gamma` to 25.0, the slider will be clamped to maxvalue (20) but the actual gamma will be 2.5. The UI will be out of sync with reality.

3. **Unconditional Cache on Init**: `UI_DisplayOptionsMenu_Cache()` is called every time the menu is opened, not just once at module load. Shader pre-registration is idempotent, so this is safe, but it's slightly inefficient. (The renderer likely deduplicates registration requests, so the cost is only a hash-table lookup per open, which is negligible.)
