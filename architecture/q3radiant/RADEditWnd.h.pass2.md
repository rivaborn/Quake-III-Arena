# q3radiant/RADEditWnd.h — Enhanced Analysis

## Architectural Role

`RADEditWnd` is a lightweight MFC container window for the level editor's script/entity editor view. It bridges the Q3Radiant document/view architecture with platform-specific rich-text editing via the wrapped `CEditWnd`. This file belongs to the **editor toolchain** (not the runtime engine); it participates in the level authoring pipeline that *generates* data consumed by the runtime BSP/AAS systems.

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main frame hierarchy**: Parent class `CWnd` is MFC's window base; instantiated by frame/dialog classes managing editor panels
- **EditWnd.h**: Direct inclusion dependency on `CEditWnd`, the actual text-editing widget
- Message dispatch: `OnCreate`/`OnSize` handlers respond to Windows GUI events from the MFC message pump

### Outgoing (what this file depends on)
- **EditWnd.h** (custom wrapper around MFC `CEdit` or rich text control)
- **MFC framework** (`CWnd`, AFX macros for message routing)
- **Windows SDK** (implicit via `LPCREATESTRUCT`, `UINT`)
- No dependencies on runtime engine (`qcommon`, `renderer`, etc.)

## Design Patterns & Rationale

**Composition over direct inheritance**: Rather than directly inheriting from `CEdit`, `RADEditWnd` composes a `CEditWnd` member. This allows:
- Custom initialization/behavior in `OnCreate` (e.g., font setup, syntax highlighting prep)
- Deferred resizing logic in `OnSize` to reflowed child layout
- A clean interface (`GetEditWnd()`) that hides implementation details

**MFC message-map pattern**: The `DECLARE_MESSAGE_MAP()` and `afx_msg` handlers are boilerplate Windows/MFC conventions—the framework routes `WM_CREATE` and `WM_SIZE` to these handlers automatically. This is idiomatic to mid-1990s Windows UI frameworks (no modern event delegation).

## Data Flow Through This File

1. **Instantiation**: Q3Radiant frame creates `CRADEditWnd` as a docking panel or dialog child
2. **Window creation**: Windows OS sends `WM_CREATE` → `OnCreate()` → initializes `m_wndEdit` child window
3. **User interaction**: Text editing happens in `m_wndEdit`; user resizes frame → `WM_SIZE` → `OnSize()` adjusts child window geometry
4. **Query**: External code calls `GetEditWnd()` to access the editor for script/entity data serialization

## Learning Notes

**Editor architecture separation**: Q3Radiant is a completely separate codebase from the runtime engine. Its classes use Windows-only patterns (MFC, `CWnd`, message maps) that would never appear in the portable `code/qcommon`, `code/client`, or `code/server` subsystems. This reflects the era's tool/engine split: tools were Windows-only, engines were cross-platform (Linux, macOS support in `code/unix`, `code/macosx`).

**Mid-1990s UI convention**: MFC message routing via `AFX_VIRTUAL` and `DECLARE_MESSAGE_MAP()` is characteristic of Visual C++ 4–6 era (1995–2000). Modern UI frameworks use event delegation or data binding instead. The `dynamic_cast<CEdit*>(&m_wndEdit)` suggests `CEditWnd` may inherit from `CEdit` or wrap it polymorphically.

**Container/adapter pattern**: `RADEditWnd` is a thin adapter—it delegates most functionality to `m_wndEdit` while providing window-level lifecycle (create, resize). This allows `EditWnd` to focus on text semantics (syntax highlighting, script parsing) rather than OS windowing.

## Potential Issues

- **No explicit error handling** in `OnCreate` or `OnSize`. If `m_wndEdit` construction fails, the window may be left in an inconsistent state.
- **GetEditWnd() returns a `CEdit*` via `dynamic_cast`**: If `m_wndEdit` does not inherit from `CEdit` (only wraps it), this returns `nullptr` silently, leading to hard-to-debug crashes if callers assume a non-null result.
