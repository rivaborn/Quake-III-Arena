# q3radiant/ZView.cpp — Enhanced Analysis

## Architectural Role

`ZView` is a viewport class in the Q3Radiant level editor implementing one of the standard orthographic map-editing views—specifically the Z-axis (elevation/side) perspective. Unlike the runtime renderer in `code/renderer/`, which produces a single 3D GL output, Q3Radiant splits the map into multiple coordinated views (XY plan, Z elevation, 3D camera) so mappers can edit from multiple angles. ZView participates in this multi-viewport editor UI coordinated through MFC's Document/View architecture, where each viewport window observes the same underlying map document and must reflect user edits consistently.

## Key Cross-References

### Incoming (what references this file)
- **MFC Document/View Framework**: `CZView` is instantiated via `IMPLEMENT_DYNCREATE` reflection, likely from a document template or splitter window. The parent `CDocument* pDoc` obtained in `OnDraw()` connects it to the shared map model.
- **Window system (implicit)**: Message handlers (create, destroy, size, input) are routed by the MFC message map (`BEGIN_MESSAGE_MAP`), which is how the OS/framework calls into this class. No explicit caller visible in code.
- **Related viewport classes**: Architecturally paralleled by `XYWnd` (XY orthographic), `CamWnd` (3D camera), and `ToolWnd` (properties/toolbar), which share the same document and coordinate viewport state through document notifications or a shared editor state object.

### Outgoing (what this file depends on)
- **MFC base classes**: `CView` (inherited); `CDocument`, `CDC`, `MINMAXINFO`, `CPoint`—all standard Windows GUI primitives.
- **Radiant.h (implicit)**: Includes `stdafx.h` and `Radiant.h`, suggesting inclusion of application-wide configuration, macro definitions, and shared document class (likely `RadiantDoc.h`).
- **Platform**: Windows/Win32 API (via MFC) for window messages and device context drawing.

## Design Patterns & Rationale

### Multi-Document Interface (MDI) with Split Viewports
The design splits a single map document across multiple simultaneous views:
- **ZView**: Z-axis orthographic (side/elevation view)
- **XYWnd** (inferred): XY-axis orthographic (plan/top view)
- **CamWnd**: 3D perspective camera
- **ToolWnd**: Properties inspector

Each inherits from `CView` and receives `OnUpdate()` notifications when the document changes, allowing true WYSIWYG multi-perspective editing—the foundational pattern in 3D level editors like QuArK, NetRadiant, and Radiant itself.

### Message-Driven Input Processing
The message map routes OS events (`WM_CREATE`, `WM_KEYDOWN`, `WM_LBUTTONDOWN`, `WM_MOUSEMOVE`, etc.) to individual handlers. This is idiomatic MFC but represents a higher-latency input pipeline compared to modern event queues—each message incurs a virtual dispatch and stack overhead. The handlers are currently stubbed with `TODO` comments, suggesting the Z-view input integration was not completed.

### Lazy Implementation
The presence of many handler stubs with TODO comments suggests the class was scaffolded but never fully implemented. This is common in shipped tools where viewport features are drafted but deprioritized.

## Data Flow Through This File

1. **Initialization** (`OnCreate`): Window is created; no custom setup visible (stub).
2. **Input Events**: OS → MFC message dispatch → handler (e.g., `OnLButtonDown`, `OnMouseMove`, `OnKeyDown`).
3. **Rendering** (`OnDraw`): Called by MFC when the window needs redrawing; receives a device context (`CDC*`) but is currently empty (TODO).
4. **Window Lifecycle**: `OnSize` for resize handling, `OnDestroy` for cleanup (both stubbed).

The lack of implementation suggests **no actual data flows through this file in the shipped build**—the viewport is essentially inert.

## Learning Notes

### Multi-Viewport Editing (vs. Monoculture Rendering)
Modern game engines (e.g., Unreal, Unity) integrate their editors within monolithic applications with immediate-mode GUIs (Dear ImGui, Qt). Q3Radiant predates this; it uses retained-mode MFC windows, which split the workload into decoupled viewports. **Lesson**: The architectural separation is elegant (each view is independent) but the scaffolding burden is high—compare the three lines of message handlers here to implementing a single unified 3D viewport.

### MFC vs. Modern GUI
This code is quintessentially early-2000s Windows C++:
- **Message maps**: Compile-time macros that generate reflection.
- **Document/View pattern**: Pre-dates modern data-binding and reactive frameworks.
- **NO modern abstractions**: No scene graph, no retained command buffers, no threading—just synchronous draw-on-demand.

**Era signifier**: The hardcoded `640×480` UI coordinate space (inferred from UI VM code) and lack of DPI awareness signal this was written before high-DPI monitors became standard.

## Potential Issues

1. **Unfinished Implementation**: All input and draw handlers are stubbed. The Z-view likely never functioned as a fully interactive viewport—users would map using only XY + camera views.
2. **Missing Synchronization**: No visible coordination mechanism between this viewport and others (e.g., when user pans/rotates the camera, should Z-view scroll?). This would normally be handled by document change notifications or a shared viewport state object (not visible in this file).
3. **Memory/Resource Leaks**: `OnDestroy` is empty—if this viewport allocated any resources in `OnCreate` or elsewhere, they would leak. Current stub code is safe only because it's vacuous.
4. **No Error Handling**: CDC operations in `OnDraw` would fail silently; in shipped code, even stub rendering requires defensive checks.

---

**Summary**: ZView is a scaffolded orthographic editor viewport that was never completed. Its architectural role is clear (Z-axis map editing), but the implementation is a hollow MFC skeleton. The fact that three other viewport types exist (`XY`, `Camera`, `Tool`) in the same codebase confirms the multi-view pattern; ZView's incompleteness suggests it was either deprioritized or superseded by the more feature-complete perspectives.
