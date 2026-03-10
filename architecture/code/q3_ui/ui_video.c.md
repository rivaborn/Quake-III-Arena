# code/q3_ui/ui_video.c

## File Purpose
Implements two UI menus for Quake III Arena: the **Driver Info** screen (read-only display of OpenGL vendor/renderer/extension strings) and the **Graphics Options** screen (interactive controls for video settings such as resolution, color depth, texture quality, and geometry detail).

## Core Responsibilities
- Build and display the Driver Info menu, parsing and rendering GL extension strings in two columns
- Build and display the Graphics Options menu with spin controls, sliders, and bitmaps for all major renderer cvars
- Apply pending video changes by writing renderer cvars and issuing `vid_restart`
- Track initial video state (`s_ivo`) to determine when the "Apply" button should be shown
- Match current settings against predefined quality presets (High/Normal/Fast/Fastest/Custom)
- Navigate between sibling option menus (Display, Sound, Network) via tab-style text buttons
- Preload all UI art shaders via cache functions

## Key Types / Data Structures

| Name | Kind | Purpose |
|---|---|---|
| `driverinfo_t` | struct | State for the Driver Info menu: menu framework, banner, frame bitmaps, back button, tokenized extension string buffer |
| `graphicsoptions_t` | struct | State for the Graphics Options menu: all menu items (spincontrols, slider, bitmaps, nav tabs) |
| `InitialVideoOptions_s` | struct | Snapshot of video settings at menu open time; used to detect dirty state and drive the Apply button visibility |

## Global / File-Static State

| Name | Type | Scope | Purpose |
|---|---|---|---|
| `s_driverinfo` | `driverinfo_t` | static (file) | Singleton instance of the Driver Info menu |
| `s_graphicsoptions` | `graphicsoptions_t` | static (file) | Singleton instance of the Graphics Options menu |
| `s_ivo` | `InitialVideoOptions_s` | static (file) | Baseline video state captured at menu init; dirty-check reference |
| `s_ivo_templates` | `InitialVideoOptions_s[5]` | static (file) | Five fixed quality presets (High Quality → Fastest + Custom) |
| `s_drivers` | `const char*[]` | static (file) | GL driver name strings indexed by driver spin-control value |

## Key Functions / Methods

### DriverInfo_MenuDraw
- **Signature:** `static void DriverInfo_MenuDraw(void)`
- **Purpose:** Custom draw callback for the Driver Info menu; renders vendor, version, renderer, pixel format, and extension strings.
- **Inputs:** None (reads `s_driverinfo`, `uis.glconfig`)
- **Outputs/Return:** void
- **Side effects:** Issues `UI_DrawString` render calls
- **Calls:** `Menu_Draw`, `UI_DrawString`, `va`
- **Notes:** Extensions are rendered in two columns; odd-count trailing extension is centered.

### UI_DriverInfo_Menu
- **Signature:** `static void UI_DriverInfo_Menu(void)`
- **Purpose:** Initializes and pushes the Driver Info menu onto the menu stack.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Writes `s_driverinfo`; calls `UI_PushMenu`; tokenizes `uis.glconfig.extensions_string` in-place into `s_driverinfo.stringbuff`
- **Calls:** `DriverInfo_Cache`, `Q_strncpyz`, `Menu_AddItem`, `UI_PushMenu`
- **Notes:** Extension string is capped at 1024 bytes and at most 40 tokens to avoid overflow (bug #399 workaround). Strings longer than 32 chars get a trailing `>` truncation marker.

### GraphicsOptions_ApplyChanges
- **Signature:** `static void GraphicsOptions_ApplyChanges(void *unused, int notification)`
- **Purpose:** Callback invoked when the Apply bitmap is activated; writes all graphics cvars and restarts the renderer.
- **Inputs:** `notification` — must be `QM_ACTIVATED`
- **Outputs/Return:** void
- **Side effects:** Calls `trap_Cvar_SetValue`/`trap_Cvar_Set` for `r_texturebits`, `r_picmip`, `r_allowExtensions`, `r_mode`, `r_fullscreen`, `r_glDriver`, `r_colorbits`, `r_depthbits`, `r_stencilbits`, `r_vertexLight`, `r_lodBias`, `r_subdivisions`, `r_textureMode`; issues `vid_restart` via `trap_Cmd_ExecuteText`
- **Calls:** `trap_Cvar_SetValue`, `trap_Cvar_Set`, `trap_Cmd_ExecuteText`

### GraphicsOptions_UpdateMenuItems
- **Signature:** `static void GraphicsOptions_UpdateMenuItems(void)`
- **Purpose:** Enforces UI constraints (3Dfx forces fullscreen/16-bit) and controls Apply button visibility by comparing current values against `s_ivo`.
- **Inputs:** None
- **Outputs/Return:** void
- **Side effects:** Modifies `flags` and `curvalue` fields on several menu items in `s_graphicsoptions`
- **Calls:** `GraphicsOptions_CheckConfig`
- **Notes:** Called every frame from `GraphicsOptions_MenuDraw`; the comment `//APSFIX - rework this` indicates known tech debt.

### GraphicsOptions_CheckConfig
- **Signature:** `static void GraphicsOptions_CheckConfig(void)`
- **Purpose:** Scans `s_ivo_templates` to find a matching preset; sets `s_graphicsoptions.list.curvalue` accordingly (defaults to index 4 = Custom).
- **Notes:** `texturebits` comparison is commented out, so it is intentionally excluded from preset matching.

### GraphicsOptions_SetMenuItems
- **Signature:** `static void GraphicsOptions_SetMenuItems(void)`
- **Purpose:** Reads renderer cvars and populates menu item `curvalue` fields before display.
- **Calls:** `trap_Cvar_VariableValue`, `UI_Cvar_VariableString`, `Q_stricmp`

### GraphicsOptions_MenuInit
- **Signature:** `void GraphicsOptions_MenuInit(void)`
- **Purpose:** Allocates and fully initializes the Graphics Options menu: layout, item definitions, initial values, and driver-type-specific hiding.
- **Side effects:** Resets `s_graphicsoptions`; calls `GraphicsOptions_SetMenuItems` then `GraphicsOptions_GetInitialVideo`; may hide driver spincontrol for 3Dfx ICD hardware.
- **Calls:** `GraphicsOptions_Cache`, `Menu_AddItem`, `GraphicsOptions_SetMenuItems`, `GraphicsOptions_GetInitialVideo`

### UI_GraphicsOptionsMenu
- **Signature:** `void UI_GraphicsOptionsMenu(void)`
- **Purpose:** Public entry point; initializes and pushes the Graphics Options menu, setting cursor to the GRAPHICS tab.
- **Calls:** `GraphicsOptions_MenuInit`, `UI_PushMenu`, `Menu_SetCursorToItem`

## Control Flow Notes
- Both menus are **push/pop** driven: `UI_GraphicsOptionsMenu` pushes onto the menu stack; the Back button calls `UI_PopMenu`.
- `GraphicsOptions_MenuDraw` is set as the menu's `draw` callback, called each frame by the menu system via `Menu_Draw`.
- `GraphicsOptions_UpdateMenuItems` is polled every frame in the draw callback — dirty state detection is continuous, not event-driven.
- `GraphicsOptions_ApplyChanges` triggers a full engine restart (`vid_restart`), making it a deferred, destructive operation from the UI's perspective.

## External Dependencies
- `ui_local.h` — menu framework types, `uis` global (`uiStatic_t`), all `trap_*` syscalls, `UI_Push/PopMenu`, `UI_DrawString`, color constants
- `uis.glconfig` (`glconfig_t`) — GL vendor/renderer/version/extensions strings, driver type, hardware type, color/depth/stencil bits
- External menu functions: `Menu_Draw`, `Menu_AddItem`, `Menu_SetCursorToItem`
- External navigation targets (defined elsewhere): `UI_DisplayOptionsMenu`, `UI_SoundOptionsMenu`, `UI_NetworkOptionsMenu`
- Renderer cvars written: `r_mode`, `r_fullscreen`, `r_colorbits`, `r_depthbits`, `r_stencilbits`, `r_texturebits`, `r_picmip`, `r_vertexLight`, `r_lodBias`, `r_subdivisions`, `r_textureMode`, `r_allowExtensions`, `r_glDriver`
- `OPENGL_DRIVER_NAME`, `_3DFX_DRIVER_NAME` — defined elsewhere (platform headers)
