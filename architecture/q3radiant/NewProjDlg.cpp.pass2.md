# q3radiant/NewProjDlg.cpp — Enhanced Analysis

## Architectural Role

This file implements a modal Windows dialog for creating new Radiant projects. It belongs to the **offline level editor toolchain** (`q3radiant/`), not the runtime engine. The dialog is part of the MFC-based GUI layer that bridges user intent (project creation) to the map file initialization logic in the broader Radiant framework. Its role is purely UI binding—to collect and validate a project name input before the editor instantiates the project data model.

## Key Cross-References

### Incoming (who depends on this file)
- **MainFrm.cpp / Radiant.cpp**: The main application window/frame would instantiate `CNewProjDlg` via modal dialog launch (e.g., `DoModal()`) in response to File→New Project menu action
- **Radiant.h**: The main application header likely forward-declares or includes this dialog class definition
- **UI framework (MFC)**: The dialog derives from `CDialog`, inheriting message routing and window lifecycle from the framework

### Outgoing (what this file depends on)
- **MFC framework**: `CDialog`, `CWnd`, `CDataExchange`, `DDX_Text` (MFC UI binding macros)
- **Radiant.h / stdafx.h**: Application-wide includes and resource IDs (e.g., `IDC_EDIT_NAME` control ID, `CNewProjDlg::IDD` dialog resource ID)
- **Windows SDK**: Underlying Win32 window creation and message dispatching
- No dependencies on runtime engine (`qcommon/`, `renderer/`, etc.) — this is a pure offline tool

## Design Patterns & Rationale

**Classic MFC Dialog Pattern** (pre-2000s Windows development):
- Constructor initializes member variables via `//{{AFX_DATA_INIT}}` markers (ClassWizard-generated code)
- `DoDataExchange()` binds dialog controls ↔ member variables bidirectionally (push on `ShowWindow()`, pull on `OK` click)
- Message map (`BEGIN_MESSAGE_MAP`) is empty because the dialog has no custom command handlers; all interaction is data binding

**Why structured this way**: MFC ClassWizard generated much of this boilerplate. The dialog is intentionally minimal—it delegates validation and project creation to the caller (likely in `MainFrm.cpp` or a project manager class that owns `m_strName` after `DoModal()` returns `IDOK`).

## Data Flow Through This File

1. **User action**: File → New Project menu triggers `CNewProjDlg::DoModal()` from caller
2. **Dialog init**: Constructor sets `m_strName = _T("")` (empty Unicode string)
3. **User input**: User types project name in IDC_EDIT_NAME control
4. **Validation point** (implicit): MFC validates via `DoDataExchange()` on `WM_OKPRESSED` — transfers control text → `m_strName`
5. **Caller retrieval**: Caller checks `if (dlg.DoModal() == IDOK) { use dlg.m_strName; }` to get validated name
6. **Project creation**: Name flows to project initialization logic (not shown in this file)

## Learning Notes

**Idiomatic to this era / engine**:
- This is **pure MFC** — Microsoft's C++ framework for Win32 GUI (1990s–early 2000s idiom)
- The `//{{}}` delimiters are ClassWizard markers (now obsolete) that automated dialog code generation
- `_T("")` macro (text macros) handled both ANSI and Unicode builds transparently
- **No input validation in the dialog itself** — validation logic lives elsewhere (Separation of Concerns, although loosely enforced)

**Modern engines would do differently**:
- Cross-platform UI frameworks (Qt, wxWidgets, Electron) instead of Win32-specific MFC
- Data binding via signals/slots or reactive patterns, not manual `DoDataExchange()`
- Inline validation with real-time error feedback
- Decoupling of dialog presentation from data model (MVP/MVVM)

**Connection to broader Radiant**:
- This dialog is part of the **file/project lifecycle management** tier of the editor
- It feeds into the level-editing pipeline that ultimately generates `.map` files (BSP source) consumed by `q3map` compiler and `bspc` AAS tool
- The architecture diagram shows `q3radiant/` as part of the **offline toolchain**, not the engine core—so no runtime impact

## Potential Issues

- **No validation**: Dialog accepts any string (including empty or invalid filename characters) — validation must occur in the caller or project manager. Risk: silent failure if `m_strName` is not sanitized downstream.
- **MFC deprecation**: MFC is no longer actively developed. Porting Radiant to modern platforms would require replacing this entire UI layer.
- **Resource ID fragility**: Assumes `IDC_EDIT_NAME` control ID is correctly defined in the `.rc` resource file and matches the dialog template — a mismatch would cause silent `DDX_Text()` failure.
