# q3radiant/NameDlg.cpp — Enhanced Analysis

## Architectural Role

This file implements a simple modal dialog component in **Q3 Radiant**, the standalone level editor tool for Quake III Arena. Radiant is entirely separate from the runtime engine (`code/` directory); it produces `.map` source files and compiled `.bsp` binaries that the engine (`code/qcommon/cm_load.c`, `code/renderer/tr_bsp.c`) later consumes. NameDlg is a utility dialog factored for use across Radiant's entity/brush editing subsystems wherever string input is needed (entity naming, group naming, path dialogs).

## Key Cross-References

### Incoming (who depends on this file)
- Various entity/object editors in `q3radiant/` (entities, brushes, groups) that need to prompt users for names
- Part of Radiant's MFC-based dialog infrastructure alongside `NameDlg.h`, `DialogInfo.cpp`, `ScaleDialog.cpp`, etc.
- Called through MFC modality system; parent window provided at instantiation time

### Outgoing (what this file depends on)
- MFC framework (`CDialog`, `CWnd`, message maps, `DDX_Text`)
- Windows API via MFC wrappers
- Resource system via `IDD` (dialog template ID) defined in `NameDlg.h`
- No dependencies on runtime engine or game code

## Design Patterns & Rationale

**MFC Modal Dialog Pattern:** Constructor captures initial data (`pName` → `m_strCaption`); `DoDataExchange` syncs between UI controls and member variables; `OnOK` triggers the exchange. This avoids exposing internal state and provides a clean input/output semantics—caller constructs dialog, calls `DoModal()`, reads `m_strName` on success.

**Why this structure?** MFC's DDX framework was the idiomatic Windows UI pattern in the early 2000s. Separating initialization (`pName` caption) from data exchange (`m_strName` edit control) kept dialogs reusable and testable by different callers.

## Data Flow Through This File

1. **Construction:** Caller (entity browser, brush editor, etc.) instantiates `CNameDlg` with a caption string describing what name is being requested.
2. **Display:** MFC runtime displays dialog template (IDC_EDIT_NAME control), window title set to caption in `OnInitDialog`.
3. **User Input:** User types into the edit control; MFC maintains no validation at this level.
4. **Submission:** User clicks OK → `OnOK()` → base class `CDialog::OnOK()` calls `DoDataExchange(TRUE)` → `DDX_Text` copies control text into `m_strName`.
5. **Return:** Modal dialog closes; caller reads `m_strName` to retrieve the result.

## Learning Notes

**Era-appropriate UI design:** This represents early-2000s Windows development. Modern game engines use immediate-mode UI (Dear ImGui style) or data-driven widget systems; Radiant uses retained-mode MFC dialogs with declarative layout (`.rc` resource files). The separation reflects the tool-vs-runtime distinction: editors prioritize developer convenience (native OS dialogs, forms); engines optimize for real-time rendering and player experience.

**Idiomatic Q3 patterns:** Like the rest of Radiant, this file uses MFC conventions throughout (message map macros `{{AFX_*}}`, `DDX_*` helpers). No shared code with the runtime engine—`bg_lib.c`, `q_math.c`, etc. are game-only.

## Potential Issues

- **No input validation:** `OnOK()` does not check for empty strings or invalid characters. Callers must validate the result.
- **Simplified error handling:** Modal dialogs don't recover from user cancellation elegantly in larger workflows.
- **Windows-only:** MFC is Windows-specific; Radiant never shipped on Linux/macOS in this codebase era (`q3radiant/` exists only; no equivalent in `code/unix/`, `code/macosx/`).
