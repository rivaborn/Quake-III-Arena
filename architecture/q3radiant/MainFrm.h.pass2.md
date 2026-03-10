# q3radiant/MainFrm.h — Enhanced Analysis

## Architectural Role

MainFrm.h defines the root application window for the Q3Radiant level editor—a Windows/MFC-based UI container that orchestrates all viewport windows (orthographic XY/XZ/YZ projections and 3D camera preview), toolbars, menus, and status display. Unlike the runtime engine subsystems documented in the architecture context, this file has **zero runtime impact**: it is exclusively a development tool for content creators. The class bridges the MFC windowing framework to application-specific viewport and editing logic, managing input routing, command dispatch (via ~200 menu message handlers), and plugin extensibility.

## Key Cross-References

### Incoming (who depends on this)
- `Radiant.cpp` (the MFC application class, `CRadiantApp`) instantiates and owns `CMainFrame`
- Child windows (`CXYWnd`, `CZWnd`, `CCamWnd`, `CTexWnd`, `CRADEditWnd`) emit `WM_PARENTNOTIFY` messages reflected back via `OnParentNotify`
- Plugin system (`CPlugInManager`) calls `AddPlugInMenuItem` and `CleanPlugInMenu` to dynamically register UI commands
- Dialog classes (e.g., `GroupDlg`) and tool windows reference `CMainFrame` pointers to query viewport/camera state

### Outgoing (what this file depends on)
- **MFC framework**: `CFrameWnd`, `CStatusBar`, `CDialogBar`, `CSplitterWnd`, `CMenu`, `CCmdUI`
- **Viewport windows**: owns and creates `CXYWnd` (XY/XZ/YZ orthographic views), `CCamWnd` (3D preview), `CTexWnd` (texture browser), `CZWnd`, `CRADEditWnd`
- **UI framework classes**: `CLstToolBar` (custom toolbar), `CTextureBar`, `CPlugInManager`
- **Command routing**: Windows message map macros; keyboard interception via `HandleKey`, `OnKeyDown`, `OnKeyUp`, `OnSysKeyDown`
- **Local state queries**: `Cvar_*` and command execution (implied via `UpdateStatusText`, `SetGridStatus`, etc.) delegate to the global qcommon command/cvar system, but this coupling is **indirect** through child windows

## Design Patterns & Rationale

1. **MFC Message Map Pattern**: ~150+ `afx_msg void On*()` handlers implement the classic MFC menu command dispatch. This is era-appropriate for early-2000s Windows development; menu IDs map statically to handler methods.

2. **Splitter-Based Layouts**: Three `CSplitterWnd` members (`m_wndSplit`, `m_wndSplit2`, `m_wndSplit3`) enable the famous "quad-split" viewport layout (3 orthographic + 1 camera window). `OnCreateClient` orchestrates layout; the design is **not DPI-aware** and window splitting is **non-persistent** across sessions.

3. **Centralized Input Routing**: `HandleKey` wraps keyboard events, routing them to `OnKeyDown`/`OnKeyUp` for menu item key binding display and global command dispatch. This bypasses normal MFC accelerator tables, enabling dynamic keybinding at runtime (`LoadCommandMap`, `ShowMenuItemKeyBindings`).

4. **Timer-Driven Polling Loop**: `OnTimer` → `RoutineProcessing` replaces a true game loop; the window timer fires at irregular intervals to drive viewport updates and UI refresh. This is fundamentally different from the engine's frame-locked 60+ Hz loop.

5. **Active Viewport Pattern**: `m_pActiveXY` tracks which orthographic view has focus; `SetActiveXY` manages toggle of the `SetActive` flag to highlight the active viewport.

6. **Plugin Injection Model**: `CPlugInManager` and `OnPlugIn` allow third-party DLLs to register menu items that invoke plugin callbacks. The plugin menu is cleared and rebuilt dynamically (`CleanPlugInMenu`/`AddPlugInMenuItem`), a flexible but runtime-expensive pattern.

## Data Flow Through This File

1. **Input flow**: User keyboard/mouse → MFC message → `OnKeyDown`/`OnSysKeyDown`/menu command → mapped `On*()` handler → child window state mutation or engine syscall delegation
2. **Viewport state**: Child windows (`CXYWnd`, `CCamWnd`) store rendering state; `CMainFrame` reads state via getters (`GetXYWnd`, etc.) for command context
3. **Status display**: Commands like `OnGridToggle` or `OnSnaptogrid` update `m_strStatus[15]` string array → `SetStatusText` → status bar refresh
4. **Plugin callback**: Plugin menu item selected → `OnPlugIn(nID)` → `CPlugInManager::DispatchCommand(nID)` → plugin DLL callback
5. **Routing to engine**: Most commands do **not** directly call the engine; instead they mutate child window state or trigger viewport refreshes. The actual world mutation happens in child window classes (e.g., `CXYWnd` brush manipulation, `CRADEditWnd` entity editing).

## Learning Notes

**Idiomatic to mid-2000s Windows/MFC editors:**
- **No abstraction over the Windows SDK**; heavy reliance on MFC macros and message maps
- **Monolithic window hierarchy**: A single frame window manages ~6 child windows; no modern compositor or document-view separation
- **No undo/redo in the frame** (`OnEditUndo`, `OnEditRedo` exist but are likely forwarded to active document/child window)
- **Toolbar/menu/status bar dualism**: toolbars are separate `CLstToolBar` objects, not integrated into a modern ribbon UI
- **No keyboard accessibility patterns**: keybindings are stored in a global map loaded at startup, not a dockable preferences panel

**Comparison to modern engines:**
- Modern editors (Unreal, Unity) use **retained-mode scene graphs** or **immediate-mode (ImGui) UI**; Radiant uses **immediate-mode viewport rendering** (OpenGL) but **retained-mode UI** (MFC windows)
- **No hot-reload**: plugins are loaded once at startup; modern tools support runtime recompilation
- **Single-threaded**: all UI and viewport updates on the main thread; modern editors multi-thread rendering and viewport updates
- **No spatial coherence queries**: window splitting is hard-coded; modern editors allow arbitrary view arrangements and would use a layout tree or docking system

**Scene graph pattern absence**: The child windows (`CXYWnd`) do NOT maintain a persistent entity tree; instead, they re-query the active map document on every render. This is simple but inefficient for large worlds.

## Potential Issues

1. **Keyboard event routing ambiguity**: `HandleKey` duplicates some MFC accelerator logic; if a menu item has both an accelerator and a dynamic keybinding, **precedence is undefined**.

2. **Plugin memory safety**: `m_PlugInMgr` and `m_nNextPlugInID` store plugin state, but there is **no visible plugin lifecycle management** in this header. If a plugin crashes or fails to unload, the main window may hold stale function pointers.

3. **Viewport state coupling**: `SetActiveXY` mutates `m_pActiveXY` but does **not** invalidate dependent UI state (toolbars, status bar may show stale grid/snap settings for the inactive viewport).

4. **Timer-driven refresh**: `OnTimer` → `RoutineProcessing` may not fire reliably on a heavily loaded system, causing viewport lag or missed user input feedback.

5. **No persistence of window layout**: `m_nCurrentStyle` tracks view mode (quad-view, single, etc.) but is **not serialized** to the project or registry; users must reconfigure the layout on every editor restart.
