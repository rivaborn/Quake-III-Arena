# q3radiant/RadiantDoc.cpp — Enhanced Analysis

## Architectural Role

`CRadiantDoc` is the **MFC document model** for the Q3Radiant level editor, holding the implicit state of the currently-loaded map as a C++ object. In the MFC SDI framework (inherited from `CDocument`), it serves as the central hub through which the editor's views, frame window, and tool windows access the active map. Though the actual map data structures and operations are delegated to helper modules (parsing, entity/brush manipulation, etc.), `CRadiantDoc` anchors the document lifecycle: creation, persistence serialization, and destruction of whatever edit session the user is working with.

## Key Cross-References

### Incoming (who depends on this file)
- **Main app class** (`Radiant.cpp`) — creates/destroys documents via MFC's `CWinApp::OnFileNew` / template machinery
- **Main frame window** (`MainFrm.cpp`) — queries active document to update menu state, undo/redo, and tool UI
- **View classes** (`RadiantView.cpp`, `CamWnd.cpp`, `EditWnd.cpp`, `TexWnd.cpp`, `ZView.cpp`, `XYWnd.cpp`) — receive document pointer; call into it to read/write map geometry and entity data
- **Tool dialogs** (entity browser, shader editor, etc.) — access document to enumerate/modify entities, textures, brushes
- **Undo/redo system** (`Undo.cpp`) — stores document state snapshots or deltas for command replay

### Outgoing (what this file depends on)
- **MFC framework** (`CDocument` base class) — message routing, save/load notification callbacks, document lifecycle
- **Radiant-specific map classes** (imported via `Radiant.h`, likely `Map.h`) — brush, entity, patch, face definitions
- **File I/O** (via `Serialize()`) — `.map` file format parser/writer (presumably in `Map.cpp`)
- **Tool modules** — brush/entity creation, selection, validation logic

## Design Patterns & Rationale

**MFC Document/View Pattern:** This class exemplifies the classic Document/View architectural separation: the document owns the persistent map state (what's loaded from disk), while views render and interact with it. This decoupling allows multiple views of the same document (XY, camera, 3D, textures).

**Stub Implementation:** The file is a **barebones scaffold** — constructor, destructor, and virtual overrides are empty or contain only TODO comments. This suggests either:
1. Early-stage development where the actual logic was planned but not yet implemented
2. The real work is done in helper classes (e.g., map manager, entity manager) not in the document itself
3. MFC's document-centric pattern was followed initially but actual operations bypass it

**SDI Assumption:** The presence of `OnNewDocument()` (SDI pattern) rather than `CWinDoc`-style multi-document implies a single active map at a time, stored in a global or accessible via `AfxGetMainWnd()`.

## Data Flow Through This File

- **Incoming:** User opens map file → `File→Open` → MFC calls `OnNewDocument()` → presumably loads `.map` from disk via `Serialize()` (currently stubbed)
- **Transformation:** Views and tools read/write map state via the document (brush/entity/patch collections)
- **Outgoing:** User saves → MFC calls `Serialize(ar.IsStoring())` → writes modified map back to `.map` file (currently stubbed)
- **Lifecycle:** Document deleted on quit or new-file action → destructor cleans up

*Note: The stubbed `Serialize()` suggests the actual load/save logic is elsewhere—likely a global map manager or separate file dialog handler.*

## Learning Notes

**MFC in 2005:** This code reflects the Windows-era MFC pattern (pre-.NET, pre-XAML). Modern C++ UI frameworks (Qt, Electron) abandoned this document-centric approach in favor of more explicit state management. The TODO comments suggest incomplete implementation—a sign of rapid prototyping or tool evolution.

**Game Editor Anatomy:** Most game editors (including Q3Radiant) do *not* faithfully follow the MFC document model; they use global or singleton map/world objects. The `CRadiantDoc` class is more ceremonial than functional—a container for MFC's machinery rather than the true model. Real map operations likely happen in `Q3::Map` or similar helper classes.

**Comparison to Modern Engines:** Unreal Editor, Unity, Godot all centralize map/scene state in a single active world/scene object without the MFC-style indirection. This is simpler and more direct.

## Potential Issues

- **Empty `Serialize()`:** The TODO comments indicate map I/O is completely unimplemented here. Either (1) the implementation is elsewhere, or (2) this is scaffolding that was never completed. This could cause silent data loss if a user tries to save without the real save logic wired in.
- **No validation/error handling:** Constructor and destructor silently succeed with no resource allocation/cleanup, suggesting the document is essentially stateless (all real state is elsewhere).
- **Disconnect from implementation:** The class structure suggests MVC design, but if the actual map logic is in globals or helper singletons, the document becomes a false contract with the framework.
