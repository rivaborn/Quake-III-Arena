# code/q3_ui/ui_video.c — Enhanced Analysis

## Architectural Role

This file implements a critical UI module within the **UI VM subsystem** (the interactive menu layer running in `code/q3_ui/`), specifically managing **video configuration and diagnostics**. It serves as the primary user-facing interface for renderer settings, bridging the menu framework to the renderer subsystem via cvar I/O. The file also acts as a diagnostic portal, exposing OpenGL capability strings to end-users for troubleshooting. Both the Driver Info and Graphics Options menus are pushed onto the **menu stack** and are siblings to Display/Sound/Network option menus, forming a cohesive settings UI.

## Key Cross-References

### Incoming (who depends on this file)

- **UI VM entry point** (`code/q3_ui/ui_main.c` or similar): calls `UI_GraphicsOptionsMenu()` and `UI_DriverInfo_Menu()` when user selects Graphics/Video menu options
- **Menu framework** (`code/q3_ui/ui_qmenu.c`): calls the `.draw` callback (`DriverInfo_MenuDraw` / `GraphicsOptions_MenuDraw`) every frame and dispatches input events to `DriverInfo_Event` / menu item callbacks
- **Menu stack** (`UI_PushMenu` / `UI_PopMenu`): this file's menus are push/pop managed; Back buttons call `UI_PopMenu()` to return to parent menu

### Outgoing (what this file depends on)

- **Renderer subsystem** (`code/renderer/`): 
  - Reads `uis.glconfig` struct (vendor string, version string, renderer string, extensions string, color/depth/stencil bits)
  - Writes renderer cvars (`r_mode`, `r_fullscreen`, `r_colorbits`, `r_depthbits`, `r_stencilbits`, `r_texturebits`, `r_picmip`, `r_vertexLight`, `r_lodBias`, `r_subdivisions`, `r_textureMode`, `r_allowExtensions`, `r_glDriver`) via `trap_Cvar_SetValue`/`trap_Cvar_Set`
  - Triggers renderer restart via `trap_Cmd_ExecuteText(EXEC_APPEND, "vid_restart")`

- **Cvar system** (`code/qcommon/cvar.c` via `trap_Cvar_*`): 
  - Reads current cvar values during menu init (`trap_Cvar_VariableValue`, `UI_Cvar_VariableString`)
  - Writes new values on Apply (`trap_Cvar_SetValue`, `trap_Cvar_Set`)

- **Menu framework** (`code/q3_ui/ui_qmenu.c`/`ui_atoms.c`):
  - `Menu_AddItem`, `Menu_Draw`, `Menu_SetCursorToItem` for menu state management
  - Navigation to peer menus (`UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`) via text button callbacks
  - String formatting (`va()`) for pixel format display

- **Shader registration** (`code/q3_ui/ui_main.c` or `code/renderer/`): 
  - Calls `trap_R_RegisterShaderNoMip()` to preload art assets (frame borders, back button, accept button graphics)

## Design Patterns & Rationale

### Singleton Pattern
Both `s_driverinfo` and `s_graphicsoptions` are file-static singletons, reset via `memset` on each menu init. This avoids dynamic allocation within the UI VM and ensures a clean state per open.

### Snapshot & Dirty-Check Pattern
The `s_ivo` (`InitialVideoOptions_s`) struct captures the baseline state when the menu opens. Every frame, `GraphicsOptions_UpdateMenuItems` compares current menu values against `s_ivo` to determine whether the Apply button should be visible/active. This **lazy dirty-state detection** avoids event handlers and is polled from the draw callback.

### Quality Preset Templates
The `s_ivo_templates[5]` array (High Quality → Fast → Fastest → Custom) encodes fixed configuration bundles. `GraphicsOptions_CheckConfig` matches the current settings against these templates to update the quality-level spinner. Notably, `texturebits` is **excluded from matching** (commented out), suggesting that texture quality is orthogonal to the preset hierarchy.

### Constraint Enforcement Pattern
`GraphicsOptions_UpdateMenuItems` enforces hardware-specific constraints: 
- **3Dfx driver** (deprecated ICD mode) forces fullscreen, 16-bit color, and hides the driver spinner
- **Windowed mode** disables color-depth selection
- **Extensions disabled** forces texture-bits to 16 or higher

This pattern uses bitwise flag manipulation (`|= QMF_GRAYED`, `&= ~QMF_GRAYED`) to enable/disable menu items based on dependent state, a common pattern in era-appropriate C UI frameworks.

### String Tokenization as Memory Optimization
The extension string is parsed in-place, overwriting spaces with null terminators to create a null-terminated array of pointers. This avoids per-token allocation (crucial in a memory-constrained QVM environment). A **workaround for bug #399** caps the buffer at 1024 bytes and 40 tokens to prevent overflow on long GL driver extension lists (e.g., GeForce 3).

## Data Flow Through This File

1. **Menu Init Phase** (`UI_GraphicsOptionsMenu` / `GraphicsOptions_MenuInit`):
   - Read current cvar values via `trap_Cvar_*`
   - Populate menu item `curvalue` fields
   - Snapshot state into `s_ivo`
   - Preload shader art via `trap_R_RegisterShaderNoMip`
   - Push menu onto stack

2. **Per-Frame Draw Phase** (`GraphicsOptions_MenuDraw` callback):
   - Call `Menu_Draw` to render all menu items
   - Call `GraphicsOptions_UpdateMenuItems` to:
     - Enforce constraints (3Dfx, windowed mode, extensions)
     - Compare current values vs. `s_ivo`
     - Conditionally show/hide Apply button
     - Match preset template and update quality spinner
   - (No explicit event loop—menu framework dispatches input)

3. **Apply Phase** (`GraphicsOptions_ApplyChanges` callback, invoked by Apply button):
   - Translate menu item values to renderer cvar settings
   - Write cvars via `trap_Cvar_SetValue`/`trap_Cvar_Set`
   - Execute `vid_restart` command via `trap_Cmd_ExecuteText`
   - **Engine-side**: renderer reloads all GL state, updates glconfig, and notifies UI of changes

4. **Driver Info Phase** (`UI_DriverInfo_Menu`):
   - Copy `uis.glconfig.extensions_string` into a writable buffer
   - Parse it into space-separated tokens
   - Render in two columns with center-justified trailing odd token

## Learning Notes

**Idiomatic patterns from the Q3A era:**
- **Cvar-driven configuration**: Unlike modern UI frameworks with data binding, Q3A exposes all settings as named cvars (e.g., `r_mode`, `r_fullscreen`). The UI reads/writes these strings/numbers, and the engine interprets them. This design predates property binding and state management frameworks.
- **QVM syscall boundaries**: All engine interactions (`trap_*`) are syscalls—explicit remote calls. The UI VM cannot read/write engine memory directly; it must request data (e.g., `uis.glconfig`) or execute commands.
- **Preset-driven settings**: Rather than hierarchical property sheets, Q3A uses a flat list of named, hardcoded quality profiles. Modern engines use scalable profiles (Low/Medium/High/Ultra) or user-driven sliders.
- **String constants and art assets**: UI strings and image paths are hardcoded as `#define` macros, not data-driven. This is typical of games from the 1999–2005 era.
- **Manual dirty-state tracking**: The `s_ivo` snapshot pattern is a precursor to modern change-detection and reactive UI frameworks.

**Connections to broader engine architecture:**
- This file is a **client-side-only** module (part of `code/q3_ui`, not shared with server or game VM).
- It depends on the **renderer subsystem** being fully initialized; `uis.glconfig` is populated at renderer startup.
- The `vid_restart` command is a **global engine command** (in `code/qcommon/cmd.c` or `code/client/cl_main.c`), not specific to video options; this file just invokes it.
- The menu framework is a **primitive immediate-mode-style** system (polling-based), not a retained-mode widget kit.

## Potential Issues

1. **Extension String Overflow (Bug #399)**: The in-place tokenization caps at 40 tokens and 1024 bytes. Modern GPUs with hundreds of extensions could overflow. A more robust solution would use dynamic allocation or linear-scan termination.

2. **Constraint Enforcement Tech Debt**: The `//APSFIX - rework this` comment in `GraphicsOptions_UpdateMenuItems` suggests the constraint logic is fragile and difficult to maintain. A declarative constraint system (e.g., rules table) would be cleaner.

3. **Unvalidated Cvar Writes**: `GraphicsOptions_ApplyChanges` writes cvars and immediately issues `vid_restart` without validating that the renderer accepted the changes. If a GPU doesn't support the requested mode (e.g., 4K on old hardware), the player may see a black screen or forced fallback without UI feedback.

4. **Preset Exclusion of texturebits**: The commented-out `texturebits` comparison in `GraphicsOptions_CheckConfig` suggests that texture quality was added post-preset-design, breaking the preset-matching contract. This could confuse users if they select "High Quality" but `texturebits` doesn't match.

5. **3Dfx Driver Handling**: The special-case logic for 3Dfx (forcing fullscreen, 16-bit) is vestigial (3Dfx hardware was discontinued ~2001). On modern systems, this code path is unreachable, but removing it could break legacy builds or historical documentation.

6. **No Async Restart**: `vid_restart` is **synchronous**—the engine reinitializes the renderer and blocks. On slow systems or in VR contexts, this could cause frame drops or disorientation. Modern engines often defer such resets or run them asynchronously.
