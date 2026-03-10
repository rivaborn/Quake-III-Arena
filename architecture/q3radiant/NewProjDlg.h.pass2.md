# q3radiant/NewProjDlg.h — Enhanced Analysis

## Architectural Role

`NewProjDlg.h` defines a simple MFC dialog for project creation in Q3Radiant, the Q3A level editor. The editor itself is a separate Windows-only tool (not part of the runtime engine in `code/`). This dialog occupies the **UI layer** of the editor stack, responsible for collecting user input (project name) before initializing a new map project.

## Key Cross-References

### Incoming (who depends on this)
- Called by **Q3Radiant main UI framework** (likely `MainFrm.cpp`, `Radiant.cpp`, or menu command handlers) when user selects "New Project"
- Dialog is instantiated and displayed modally to block until user confirms or cancels

### Outgoing (what this file depends on)
- **MFC framework** (`CDialog`, `CString`, `CWnd`) — Windows-only GUI abstraction
- **Resource system** (`IDD_DLG_NEWPROJECT` constant references a dialog resource in `Radiant.rc`)
- No direct dependencies on engine code (`code/` subsystems) — editor is completely separate

## Design Patterns & Rationale

**MFC Dialog Pattern**: Standard early-2000s Windows GUI idiom. Uses:
- `DoDataExchange` for two-way data binding between UI controls and member variables
- `DECLARE_MESSAGE_MAP()` skeleton for potential future message handling
- Auto-generated "ClassWizard" wrapper comments (lines 55–57) — editor generated this using MSVC 6.0 tooling

**Why this structure?** MFC dialogs were the dominant Windows GUI framework of that era. Kept editor UI self-contained without imposing GUI dependencies on the portable runtime engine.

## Data Flow Through This File

1. **Input**: User invokes "New Project" from menu → dialog instantiated with parent window pointer
2. **Transform**: User types project name into text field; MFC's DDX/DDV framework marshals into `m_strName`
3. **Output**: Dialog returns `IDOK` / `IDCANCEL` to caller; if OK, `m_strName` contains the project name passed to project creation logic

No validation is visible at the header level; validation likely occurs in the `.cpp` implementation via `OnInitDialog()` or custom message handlers.

## Learning Notes

- **Editor/Engine separation**: The Quake III engine carefully decouples the runtime (`code/`, `common/`) from the editor (`q3radiant/`, `q3map/`). Editor uses MFC; engine is portable C with minimal OS coupling.
- **Early 2000s Windows idiom**: MFC was peak Windows development at Q3A's release (2005). Modern editors use Qt, wxWidgets, or Electron. The `#pragma once` and MSVC version guards reflect Visual Studio 6.0 era constraints.
- **Minimal responsibility**: Dialog is purely presentational—no game logic, no file I/O, no path validation. Actual project creation happens elsewhere (likely `NewProjDlg.cpp` or a manager class).

## Potential Issues

- **No visible input validation** in header (may exist in `.cpp`). Empty or invalid project names could cause downstream failures.
- **Windows-only**: Hard MFC dependency makes this unmaintainable on cross-platform builds or modern Visual Studio versions. No macOS/Linux support.
- **Type safety**: `CString` is MFC-specific; modern code would use `std::string` or `QString`.

---

**Note**: The cross-reference index provided focuses on engine/tool subsystems (AAS, AI, BSP) and contains few editor-layer references, as expected—the editor is a self-contained auxiliary application, not part of the runtime architecture.
