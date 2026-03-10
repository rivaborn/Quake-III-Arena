# q3radiant/ZWnd.cpp — Enhanced Analysis

## Architectural Role

This file implements a viewport window for Q3Radiant's Z (elevation) orthographic view, one of three synchronized viewport panels in the level editor (alongside XY floor-plan and 3D camera views). It bridges the MFC window framework to the editor's Z-view logic, managing all UI input translation, OpenGL context lifecycle, and screen-space-to-world-space coordinate transformation. The viewport acts as a stateless presentation layer: all state lives in global `z` and `g_qeglobals` structures and persists across multiple viewport updates.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant/MainFrm.h** and main window (implicitly via MFC frame layout) — creates and owns this viewport
- **q3radiant/qe3.h** — accesses global editor state (`g_qeglobals`, `g_pParentWnd`, global `z` struct)
- No direct function calls into this file; instead, Windows message routing handles all input

### Outgoing (what this file depends on)
- **Z-view core logic** — calls `Z_MouseDown()`, `Z_MouseUp()`, `Z_MouseMoved()`, `Z_Draw()` (defined in Z.cpp, as implied by `#include "qe3.h"`)
- **Global state** — reads/writes:
  - `g_qeglobals.d_hwndZ` — registers this window handle at creation
  - `g_qeglobals.d_hglrcBase` — the shared OpenGL context created elsewhere, used for `wglShareLists`
  - Global `z` viewport (origin[2], height, scale, width, grid snapping)
  - `g_pParentWnd` — parent window proxy for status bar text and keyboard dispatch
- **OpenGL wrappers** — `qwgl*` prefixed functions (dynamic GL function pointers managed by renderer)
- **QE utilities** — `QEW_SetupPixelFormat()`, `QEW_StopGL()` from qe3.h (GL setup/teardown)
- **MFC framework** — `CWnd`, `CPaintDC`, message map macros

## Design Patterns & Rationale

### Shared OpenGL Context
```cpp
qwglCreateContext(m_dcZ)
qwglShareLists(g_qeglobals.d_hglrcBase, m_hglrcZ)
```
Multiple viewport windows share a single OpenGL context chain so they can all reference the same texture objects and display lists. This is a classic pattern in multi-view editors and avoids GPU memory duplication.

### Coordinate System Flip
```cpp
Z_MouseDown(point.x, rctZ.Height() - 1 - point.y, nFlags)
```
Windows reports mouse Y from top-down (0=top), but world coordinates go bottom-up. The flip `Height() - 1 - y` keeps the editor's internal logic orthogonal (world space), not screen space. Consistent across all three mouse button handlers.

### Grid-Snapped Status Display
In `OnMouseMove()`, the Z-coordinate is snapped to grid spacing (`g_qeglobals.d_gridsize`) before display. This is UI-only feedback; the actual movement logic in `Z_MouseMoved()` may use finer precision.

### Mouse Capture Pattern
Left, middle, and right buttons all call `SetCapture()` to continue receiving mouse events even when the pointer leaves the window. Release only when all buttons are released (checked via `nFlags & (MK_LBUTTON|MK_RBUTTON|MK_MBUTTON)`).

### Window Class Pre-Registration
`PreCreateWindow()` registers a custom window class `Z_WINDOW_CLASS` with style `CS_OWNDC` (each window gets its own device context, not shared from parent). This is necessary for independent OpenGL rendering.

## Data Flow Through This File

1. **Initialization** (`OnCreate`):
   - Register window handle in global `g_qeglobals.d_hwndZ`
   - Get device context, set up pixel format, create GL context
   - Share display lists with base context
   - Make context current

2. **User Input** (mouse/keyboard events):
   - All three mouse buttons → `Z_MouseDown()` with flipped Y coordinate
   - Mouse move → `Z_MouseMoved()` + status bar update (grid-snapped display)
   - All mouse up → `Z_MouseUp()` + conditional release capture
   - Keyboard → `g_pParentWnd->HandleKey()` (dispatch to parent)

3. **Rendering** (`OnPaint`):
   - Make GL context current (may differ from `m_dcZ` in paint handler)
   - Call `Z_Draw()` (core viewport render)
   - Swap buffers

4. **Shutdown** (`OnDestroy`):
   - `QEW_StopGL()` cleans up GL context and device context

## Learning Notes

- **Idiomatic to editor codebases of this era** (early 2000s, MFC):
  - Direct global state rather than dependency injection
  - Message map macros hide much of the routing boilerplate
  - GL context per-viewport instead of modern framebuffer-based approach (textures back each viewport)
  - Mouse capture for drag operations (predates modern pointer lock APIs)

- **Contrast with modern engines**:
  - Modern level editors use framebuffer objects (FBO) so each viewport renders to a texture, avoiding context switching
  - Coordinate flips are usually hidden in viewport/scissor rectangle setup, not in event handlers
  - MFC's message routing is now replaced by event systems or immediate-mode UI frameworks

- **Important architectural insight**: The three viewport windows (XY, Z, and 3D camera) are independent presentation layers that all read and write to a shared `z` (and equiv XY/camera) state structure. This is a classic **Observer pattern variant** — many views, one underlying model. Changes to selection or viewport pan/zoom update the shared structure; all viewports see them on next paint.

## Potential Issues

- **Y-coordinate flip inconsistency risk** (line 226): `OnLButtonUp` uses `rctZ.bottom - 1 - point.y` while `OnMouseMove` (line 164) uses `rctZ.Height() - 1 - point.y`. Both should be equivalent, but inconsistent naming obscures intent. `rctZ.bottom` directly accesses a rect member, while `Height()` is a helper; this inconsistency across move/down/up handlers could harbour subtle bugs if rect coordinates ever differ.

- **Unvalidated GL errors** (line 134): On `wglMakeCurrent` failure, the code prints an error but continues to call `Z_Draw()` with a possibly invalid context. Should either retry, skip the draw, or fail hard.

- **No error handling for `Z_Draw()`** — if the Z-view core crashes or returns an error, this code has no recovery path; the paint will fail silently or crash the editor.
