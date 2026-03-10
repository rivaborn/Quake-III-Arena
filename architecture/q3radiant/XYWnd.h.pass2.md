# q3radiant/XYWnd.h — Enhanced Analysis

## Architectural Role

**XYWnd** is the orthographic top-down (XY plane) viewport widget for the Q3Radiant level editor—one of three spatial windows (alongside **CamWnd** camera viewport and Z elevation view) enabling 3D map construction. Unlike the runtime engine subsystems (renderer, physics, networking), this is a **design-time tool** component that translates 2D mouse input into 3D brush and entity manipulation commands, then reflects changes back to the level database and other viewports. It bridges MFC UI framework to Radiant's brush-entity scene graph.

## Key Cross-References

### Incoming (what calls this)
- **MainFrm** (parent frame window) instantiates and docks XYWnd as a persistent viewport pane
- **CamWnd** (camera viewport) marked as `friend` — bidirectional viewport synchronization (e.g., orbit-follow, selection sync)
- Editor document class (`RadiantDoc`) likely updates XYWnd when brushes/entities are loaded or modified
- Message handlers route user input (mouse, keyboard) from MFC to this window's event methods

### Outgoing (what this calls)
- **qe3.h** (editor core): Contains global editor state, brush/entity registry, undo/selection state
- **CamWnd.h**: Cross-viewport updates (when user manipulates viewport scale, origin, or selects geometry)
- **brush_t** structures: Direct manipulation of brush geometry via clipping, scaling, rotation
- Renderer via qcommon collision system: For visibility testing and grid snapping (implied by `CM_*` usage in level editor context)

## Design Patterns & Rationale

1. **MFC Window Subclassing** — Inherits `CWnd` with `DECLARE_DYNCREATE` for dynamic instantiation; standard 1990s Windows GUI pattern for embedding custom views in managed frames.

2. **Mode State Machine** — Discrete editing modes (ClipMode, RotateMode, ScaleMode, PathMode, PointMode) are mutually exclusive. Each has getter/setter pairs:
   ```cpp
   bool ClipMode(); void SetClipMode(bool bMode);
   bool RotateMode(); bool SetRotateMode(bool bMode);
   ```
   This is more primitive than modern ECS or command-pattern UI toolkits but sufficient for a 1999-era editor.

3. **Helper Class for Transient State** — **CClipPoint** wraps a 3D point, its 2D screen projection, and an optional pointer to shared 3D data. This pattern allows clipboard-like clip operations without tight coupling to brush geometry.

4. **Message Map Dispatch** — MFC's `AFX_MSG` macros and `DECLARE_MESSAGE_MAP()` route Windows messages (OnLButtonDown, OnMouseMove, etc.) to handler methods. Classic inversion-of-control pattern pre-dating event listeners.

5. **Coordinate Transformation Pipeline** — Methods like `XY_ToGridPoint()` and `SnapToPoint()` encapsulate the screen→world→grid→snapped conversion, keeping viewport logic cohesive.

## Data Flow Through This File

```
User Input (Mouse/Keyboard)
    ↓
MFC Message Handler (OnLButtonDown, OnMouseMove, OnKeyDown)
    ↓
Mode-Specific Processing (DragDelta, NewBrushDrag, DropClipPoint, etc.)
    ↓
Coordinate Transforms (XY_ToPoint, SnapToPoint, grid alignment)
    ↓
Brush/Entity Manipulation (via qe3.h global state)
    ↓
Undo Record (UndoCopy, UndoAvailable)
    ↓
Viewport Redraw (Redraw, XY_Draw, XY_Overlay)
    ↓
Cross-Viewport Sync (CamWnd friend updates)
```

**Key state held locally:**
- `m_fScale` — viewport zoom level
- `m_vOrigin` — viewport pan center (world coordinates)
- `m_nViewType` — which plane (XY, XZ, YZ)
- `m_ptCursor`, `m_ptDown`, `m_ptDrag*` — transient mouse tracking
- `m_bPress_selection` — whether drag started on existing geometry
- Mode flags and clip/rotate/path point arrays

## Learning Notes

1. **Pre-Modern Editor Architecture** — This codebase predates scriptable editors (Blender, Godot) and immediate-mode UI (ImGui). It shows the "immediate retained-mode" MFC approach: state lives in window members, messages mutate it, redraw happens next frame.

2. **Orthographic Viewport Conventions** — The XY viewport is top-down (looking down Z axis). The class name itself embeds the convention; a modern engine might use generic "Viewport2D" or "EditorPane". Radiant hardcodes the axis orientation.

3. **Grid Snapping & Precision** — Methods like `XY_ToGridPoint()` enforce discrete grid alignment (visible in `XY_DrawBlockGrid()`). This was essential for 1990s polygon-based level design; modern engines often use continuous placement with optional snap-to-grid.

4. **Undo/Redo as Dual-Copy State** — `UndoCopy()` and `Paste()` suggest that the undo system stores complete snapshots rather than deltas. Contrast this with modern command-pattern undo (store only the operation and its inverse) or ECS-based undo (snapshot entity component state).

5. **Brush-Centric Workflow** — All visible operations (clipping, rotating, scaling) work on **brushes** — the fundamental geometric primitive. Entities are placed and moved but not directly modified here. This split reflects Q3's content model: brushes define solids; entities are spawners/logic nodes.

6. **Viewport Synchronization by Friendship** — The `friend CCamWnd` declaration is a code smell by modern standards (breaks encapsulation) but was pragmatic in 1999: both viewports need to coordinate (pan, zoom, selection, visible bounds). Proper solution would be a shared viewport controller or event bus.

## Potential Issues

1. **Message Handler Bloat** — The class accumulates many small handler methods (`OnLButtonDown`, `OnMButtonDown`, `OnRButtonDown`, etc.). A modern UI framework would route all input through a unified `OnInput(InputEvent)` method, reducing boilerplate and improving consistency.

2. **State Coherence Risk** — Multiple mode flags (`m_bPress_selection`, clip/rotate/scale/path/point states) are checked and set independently. If a mode change handler fails partway, the viewport can enter an inconsistent state (e.g., ClipMode true but no clip points allocated).

3. **Coordinate System Ambiguity** — Method names like `XY_ToPoint()` mix viewport-specific naming (`XY_`) with generic purpose (`ToPoint`). A reader must consult implementation to know whether it returns world-space, view-space, or grid-snapped coordinates.

4. **Friend Class Tight Coupling** — The `friend CCamWnd` decorator bypasses all encapsulation. Refactoring either class requires careful coordination; adding a third viewport becomes complicated.
