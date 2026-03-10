# q3radiant/RADEditView.h — Enhanced Analysis

## Architectural Role

`CRADEditView` is a text editor view component within Q3Radiant, the offline map editor for Quake III Arena. It provides a dual-view editing interface for map data: while the main editor (`MainFrm.cpp`, `CamWnd.cpp`, `XYWnd.cpp`) offers graphical 3D/2D manipulation, this view class surfaces the underlying entity script data as editable text. This allows mappers to directly inspect and edit `.entity` strings or shader definitions without round-tripping through the graphical interface. It sits within the MFC document-view architecture as part of the editor's multi-pane UI.

## Key Cross-References

### Incoming (who depends on this file)
- `RadiantDoc.cpp` / `RadiantView.cpp`: The MFC document-view container instantiates this view class as part of the main editor frame layout
- `ChildFrm.cpp`: Manages splitter windows and layout; creates/destroys `CRADEditView` instances per open documents
- MFC framework itself (`CEditView`): Base class that handles text buffer management, undo/redo, clipboard integration

### Outgoing (what this file depends on)
- `CEditView` (MFC): Provides all text editing infrastructure (selection, insertion, deletion, rendering via CDC)
- MFC message map system: Routes Windows messages (`WM_*`) to handler methods
- Renderer indirectly: `OnDraw` receives a device context but delegates actual rendering to `CEditView` base

## Design Patterns & Rationale

**MFC Document-View Architecture**: Q3Radiant adopts the classic MFC pattern of separating document state (`RadiantDoc`) from presentation (`CRADEditView` + graphical views). This allows multiple simultaneous views of the same map data.

**Minimal View Subclass**: The class is remarkably spare—just a constructor, destructor, `OnDraw`, and `OnChange` handler. This suggests `CEditView` already handles the heavy lifting (text rendering, input handling, clipboard). The `OnChange` likely triggers updates to the underlying document or broadcasts change notifications to sibling views.

**Paired with Graphical Views**: The architecture implies a workflow where the entity text editor (`CRADEditView`) and 3D/2D viewports (`CamWnd`, `XYWnd`) are kept in sync—a change in the text view must invalidate the 3D representation, and vice versa.

## Data Flow Through This File

1. **Inbound**: User keyboard input → MFC message dispatch → `OnChange` handler → back to `RadiantDoc`
2. **View State**: Document modification → broadcast to all open views (including this one) → `OnDraw` re-renders text
3. **Outbound**: User edits in text → propagate to other views (invalidate 3D, update entity properties)

The `OnChange` handler is the critical synchronization point—it likely notifies the parent document and possibly sibling views (e.g., entity property inspector, 3D viewport) that the entity data has changed.

## Learning Notes

**Era-Specific MFC Patterns**: This code exemplifies late-1990s Windows desktop UI patterns. Modern engines use Qt, Electron, or custom GL-based UIs; MFC's message maps and virtual function overrides are now quaint but were standard practice in that era.

**Editor Complexity**: Q3Radiant demonstrates the typical complexity of a level editor: multiple synchronized views of the same data (text, graphical 3D, 2D orthographic, entity list, shader browser, lighting preview). A single change must cascade through all views.

**Tool-Engine Separation**: Unlike modern integrated engines, Q3Radiant is a *separate* tool that generates `.map` (human-readable) and `.bsp` (compiled) files. The pipeline is: Q3Radiant (editor) → `.map` file → `q3map.exe` (BSP compiler) → `.bsp` file → runtime engine. This editor view is part of that offline toolchain.

## Potential Issues

- **Minimal Error Handling**: The header shows no validation or error callbacks in `OnChange`; if synchronization with the document fails silently, the text view and 3D viewport could drift out of sync.
- **No Observable Message Map**: The `{{AFX_MSG}}` block is a ClassWizard-generated marker. If `OnChange` is wired manually or incorrectly, edits might not propagate to the document.
- **Text-Only Representation**: Entity data in Quake III is key-value pairs and scripts; representing it as raw text without syntax highlighting or validation could lead to hard-to-debug map corruption if the user introduces malformed data.
