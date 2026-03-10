# q3radiant/XYWnd.cpp — Enhanced Analysis

## Architectural Role

This file implements the **XY (top-down) 2D viewport** of the Quake III Radiant level editor—one of three orthogonal map views (XY, XZ, YZ) plus a 3D camera view. It is responsible for rendering the 2D BSP layout, handling user input (brush selection, manipulation, clipping, pathing), and coordinate snapping. It is **entirely separate from the runtime engine** (in `code/`); Radiant is a standalone map authoring tool with no dependencies on or inclusion in shipped binaries.

## Key Cross-References

### Incoming (who depends on this file)
- **MainFrm.h / MainFrm.cpp** (editor main frame): Creates and manages CXYWnd instances; calls `SetActiveXY()` to designate the active viewport; broadcasts `Sys_UpdateWindows()` to refresh all views after edits
- **Radiant.cpp** (MFC app class): Initializes editor subsystems; Radiant.h includes XYWnd.h
- **PrefsDlg.h** (preferences dialog): Reads grid size and editor cvars (`g_nGridSize`, etc.) that affect snapping behavior in CXYWnd

### Outgoing (what this file depends on)
- **qe3.h** (main Radiant header): Global editor state (`g_qeglobals` including `d_hwndXY`, `d_hwndMain`, `d_hglrcBase` for shared GL context)
- **DialogInfo.h**: For entity/brush property dialogs
- **splines/splines.h**: Bezier curve utilities for patch editing
- **QEW_SetupPixelFormat**, **qwglCreateContext**, **qwglShareLists**, **qglPolygonStipple** (Windows GL wrappers from platform layer) for OpenGL setup
- Global brush/entity state: `g_brClipboard`, `g_brUndo`, entity_t structures (no file-level isolation; shared via extern declarations across all editor code)
- **Sys_UpdateWindows()** (qe3.cpp): Broadcasts viewport refresh commands to all open views (XY, XZ, YZ, camera)
- **SnapToPoint()** (undefined in visible code, likely in select.cpp or drag.cpp): Grid-based coordinate quantization for precise entity placement

## Design Patterns & Rationale

- **MFC Message Dispatch**: `BEGIN_MESSAGE_MAP` → `ON_WM_*` macros route Windows messages (WM_LBUTTONDOWN, WM_MOUSEMOVE, WM_PAINT) to handler methods. Idiomatic for 1990s–2000s Windows editors; modern engines use event queues instead.
  
- **Global State Machine**: Multiple `g_b*Mode` flags (`g_bClipMode`, `g_bRotateMode`, `g_bPathMode`, `g_bPointMode`) represent mutually-exclusive editor tools. A single global state variable would be more elegant; this pattern is common in legacy tool code but inflexible if multiple simultaneous tools are ever desired.

- **Immediate-Mode 2D Rendering**: No retained scene graph. OnPaint() directly issues OpenGL commands to draw grid, brushes, entity icons, clip points. Fast for small editor viewports; renderer context (`s_hdcXY`, `s_hglrcXY`) is thread-local to this window.

- **Input Capture and Dragging**: `SetCapture()` (Windows API) ensures mouse events are routed to this window even outside its rect during interactive operations like clip-point dragging. Simpler than a global event loop but tightly coupled to Win32.

- **Clipboard for Copy/Paste**: `g_brClipboard` (brush linked-list) and `CMemFile g_Clipboard` (binary serialization) implement undo/copy-paste. No abstraction layer; globals directly modified by all tools.

## Data Flow Through This File

**User Input → Editing Operations → Global State → Broadcast Refresh:**

1. **Mouse Event**: `OnLButtonDown(nFlags, CPoint point)` dispatches to one of three code paths:
   - **Clip Mode** (`g_bClipMode`): Call `DropClipPoint()` to record screen coords, snap to 3D world position via `SnapToPoint()`, store in `g_Clip1/Clip2/Clip3`
   - **Path Mode** (`g_bPathMode`): Similar; accumulate points in `g_PathPoints[]` until `g_nPathCount == g_nPathLimit`, then invoke callback `g_pPathFunc()`
   - **Default** (`OriginalButtonDown`): Likely select/deselect brushes or initiate drag operations

2. **Coordinate Snapping**: Mouse screen coords (2D pixels) are converted to 3D world coords via inverse projection matrix, then snapped to grid via `SnapToPoint()`. Result stored in clip/path point struct.

3. **Global State Update**: Brushes in `g_brClipboard`, clip points in `g_Clip1/2/3`, or point array `g_PointPoints[]` are modified.

4. **Broadcast Refresh**: Call `Sys_UpdateWindows(XY | W_CAMERA_IFON)` to tell all viewports to redraw. The XY viewport redraws on next `OnPaint()` call.

5. **Rendering**: `OnPaint()` (not shown in truncated code) iterates all visible brushes, draws them as 2D wireframe polygons in the XY plane, overlays clip points and guide geometry, calls `SwapBuffers()` to present.

## Learning Notes

- **Offline Tool Architecture**: Radiant is a **completely separate codebase** from the runtime engine. No engine APIs are called; instead, Radiant maintains its own in-memory BSP/entity representation and serializes to `.map` files that are later compiled by `q3map/` into binary `.bsp` files. This is a common pattern: build tools often are independent applications.

- **OpenGL in Dialogs**: Shows how to integrate OpenGL into a Windows dialog-based editor: `GetDC()`, `wglCreateContext()`, `wglShareLists()` to share texture/compiled lists with the main GL context, `qwglMakeCurrent()` to switch contexts before drawing.

- **Legacy Global State**: Heavy use of `extern` globals (e.g., `g_Clip1`, `g_bClipMode`) for editor tool state. Modern editors would use a command/tool manager class. This reflects 1990s C++ practice and makes refactoring difficult (no encapsulation).

- **Idiomatic Differences from Modern Engines**:
  - No **scene graph** or **entity component system (ECS)**; immediate-mode drawing each frame.
  - No **input event queue**; direct Win32 message dispatch to window class methods.
  - No **undo stack abstraction**; direct manipulation of `g_brUndo` global.
  - No **viewport/camera abstraction**; each viewport hardcodes its own projection matrix and rendering code.

## Potential Issues

- **Thread Safety**: Global state (`g_Clip1`, `g_brClipboard`, etc.) is accessed from the UI thread without locks. If any background operations (e.g., AAS compilation in `code/bspc/`) ever run concurrently, race conditions are likely. Currently not a problem because Radiant is single-threaded.

- **No Error Propagation**: `OnCreate()` calls `Error()` on GL context failure; `Error()` is a `longjmp`-based fatal exit. No graceful degradation or retry logic.

- **Hardcoded Grid/Snap Logic**: Grid size and snap behavior are embedded in the viewport; changing grid requires editing multiple XY/XZ/YZ viewport files. A shared grid manager would reduce duplication.

- **OpenGL Version Lock**: Uses OpenGL 1.x fixed-function pipeline (`qglPolygonStipple`, `qglLineStipple`); would require substantial refactor to support modern OpenGL or D3D.
