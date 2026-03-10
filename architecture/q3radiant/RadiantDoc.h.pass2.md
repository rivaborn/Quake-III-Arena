# q3radiant/RadiantDoc.h — Enhanced Analysis

## Architectural Role

`CRadiantDoc` is the MFC (Microsoft Foundation Classes) document class for the Quake III map editor, **not** part of the runtime engine. It represents the in-memory state of a map file being edited and owns the document lifecycle (new, open, save, close). The class follows the classic MFC document/view pattern where the document maintains game data and views (e.g., 3D viewport, 2D orthographic views) render it. This file is foundational infrastructure for Radiant's GUI layer—purely a development tool with no runtime dependencies.

## Key Cross-References

### Incoming (who depends on this file)
- **MainFrm.cpp** (main application window) — registers `CRadiantDoc` in the document template and manages top-level window lifecycle
- **ChildFrm.cpp** (child MDI windows) — each map editor window holds a pointer to a `CRadiantDoc` instance
- **RadiantView.cpp**, **XYWnd.cpp**, **CamWnd.cpp**, **ZWnd.cpp** — viewport classes receive document pointer and query/modify map data via the document
- **RadiantDoc.cpp** (implementation) — provides all document-level operations (dirty flag, undo/redo integration, entity list management)

### Outgoing (what this file depends on)
- **MFC base class `CDocument`** — provides serialization framework, dirty-flag management, undo/redo stack integration via message maps
- **Windows API** (indirectly through MFC) — standard Win32 document patterns
- No dependencies on `qcommon`, `engine`, `game`, or any runtime code — editor is completely isolated from runtime

## Design Patterns & Rationale

**MFC Document/View Architecture**: Standard Win32 GUI pattern circa 1998–2005. The document-view separation allows multiple viewports (3D camera, XY top-down, Z side, texture browser) to render the same underlying map simultaneously and stay synchronized.

**Serialization**: The `Serialize(CArchive& ar)` virtual override enables `CDocument::SaveModified()` and `OnOpenDocument()` to handle file I/O through a uniform stream abstraction—both reading and writing use the same code path.

**ClassWizard Generated Boilerplate**: The `//{{AFX_VIRTUAL}}` and `//{{AFX_MSG}}` markers are MFC-specific code generation directives. ClassWizard (an IDE plugin) would auto-generate message handler skeletons here; manual editing was discouraged.

**Why this design**: MFC was the dominant Windows GUI framework at the time; using it avoided writing Win32 boilerplate directly.

## Data Flow Through This File

**Loading**: User selects "Open Map" → `OnOpenDocument()` (inherited from `CDocument`) → `Serialize()` → parse `.map` file into BSP/entity data structures → views refresh via `UpdateAllViews()`.

**Editing**: Views (XYWnd, CamWnd) edit geometry/entities → call document methods → document marks itself dirty (`SetModifiedFlag()`) → views invalidate viewports.

**Saving**: User hits Ctrl+S → `OnSaveDocument()` (inherited) → `Serialize()` writes `.map` file back to disk → clears dirty flag.

**Undo/Redo**: Document maintains undo stack (via MFC's `CUndoManager` integration); view commands call `GetDocument()->RegisterUndoAction()`.

## Learning Notes

**MFC Era Architecture**: This represents mid-1990s Windows GUI patterns. Modern map editors (Unreal, Unity, Godot) use custom graphics frameworks and cross-platform C++ rather than MFC or Win32-specific patterns.

**Single-Threaded GUI**: No threading; all map editing is synchronous in the main GUI thread. The runtime engine (`code/` directory) is entirely separate and never linked into Radiant.

**Document-Centric**: Unlike the engine's highly modular VM/subsystem design, the editor bundles everything into a monolithic Win32 application. This is typical for GUI tools but would not scale to the engine's complexity.

**No Bidirectional Sync**: Radiant cannot preview the map in the runtime engine from the editor; you must export the map, run the game separately, and manually reload. This is a workflow limitation of the era.

## Potential Issues

- **No explicit threading**: File I/O in `Serialize()` blocks the GUI thread. Large maps would freeze the editor during save/load.
- **MFC version lock**: Tightly coupled to a specific MFC version; cannot easily port to modern C++ GUI frameworks.
- **Minimal header**: The public interface here is extremely thin; almost all document functionality lives in `RadiantDoc.cpp`, making this header nearly a stub.
