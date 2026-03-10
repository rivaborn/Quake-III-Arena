# q3radiant/NameDlg.h — Enhanced Analysis

## Architectural Role

`CNameDlg` is a simple modal dialog UI component in the Q3Radiant level editor, outside the runtime engine proper. It serves the editor's entity/object naming workflows, not the game engine. As a light MFC wrapper around Windows `DIALOG` resources, it bridges declarative GUI definitions (`IDD_NEWNAME` resource) to C++ event handlers. The editor instantiates this dialog when operations require a user-supplied text identifier (e.g., naming a group, brush, or entity).

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant editor UI layer** calls `CNameDlg` constructor to instantiate modally; likely invoked from menu handlers, toolbar buttons, or entity/brush creation workflows
- No runtime engine subsystems reference this file (editor tooling is entirely offline, not linked into `game.dll`, `cgame.dll`, etc.)

### Outgoing (what this file depends on)
- **MFC framework** (`CDialog`, `CString`, `CWnd`, `CDataExchange`) — Windows GUI abstraction layer
- **Windows resource system** — loads dialog template from resource ID `IDD_NEWNAME` at construction time
- **No engine dependencies** — does not call into `qcommon`, renderer, game VM, or any runtime subsystems

## Design Patterns & Rationale

**MFC message map pattern**: The `//{{AFX_MSG}}` and `DECLARE_MESSAGE_MAP()` macros are MFC's pre-C++11 event-handler registration system. Message handlers like `OnOK()` and `OnInitDialog()` are dispatched via a static message table, avoiding virtual function overhead. This is idiomatic for 1990s Windows C++ GUI code.

**Constructor-driven initialization**: The public constructor takes `const char *pName` (likely the initial/default name) and optional parent window. MFC separates construction from resource loading—`OnInitDialog()` fires after the resource template is loaded and Windows creates the HWND, allowing hydration of controls with logical data.

**Data exchange abstraction**: `DoDataExchange()` uses DDX/DDV (Data eXchange/Validation) framework to marshal between UI controls (`CString m_strName` in the dialog) and the C++ object member. This is the MFC idiom for bidirectional binding—avoids hand-coded `GetDlgItemText()`/`SetDlgItemText()` boilerplate.

## Data Flow Through This File

1. **Input**: Editor calls `CNameDlg ctor` with a candidate name string
2. **OnInitDialog()**: Fired by MFC after resource load; initializes `m_strName` from the constructor argument; populates the text control
3. **User interaction**: User edits the text field or clicks OK/Cancel
4. **OnOK()**: Fires when user clicks OK; calls `DoDataExchange()` to marshal control text back to `m_strName`; validates if needed; calls base `CDialog::EndDialog(IDOK)` to close the dialog
5. **Output**: Parent window retrieves the resulting `m_strName` value via `GetDlgItem()` or by accessing the `CNameDlg` instance's public member after `DoModal()` returns

## Learning Notes

**Offline editor vs. runtime engine**: This file illustrates the clean separation in the Quake III codebase between offline tools (`q3radiant/`, `q3map/`, `bspc/`) and the runtime engine (`code/`). The editor relies entirely on Windows/MFC and does not use engine systems—no `qcommon.h`, no VM, no network code. This makes the editor a self-contained application, not a game plugin.

**MFC idioms of the era**: The `//{{AFX_*}}` comments and `DECLARE_MESSAGE_MAP()` are specific to MFC's ClassWizard tool (Visual Studio's codegen helper circa 1990–2005). Modern Win32 or .NET would use event delegation or message hooks instead. The pattern is now archaic but was productive for its time.

**Modal dialog simplicity**: A 69-line header file encapsulates a complete modal dialog lifecycle—construction, initialization, user input, and data binding. This is intentionally lightweight compared to modern frameworks; the full implementation would be similarly sparse in `NameDlg.cpp`.

## Potential Issues

- **No validation shown**: The header exposes only interface; actual validation logic (if any) is in the `.cpp` file. If the dialog allows empty or special-character names without checking, downstream editor code may fail silently.
- **Hardcoded resource ID**: `IDD_NEWNAME` is a compile-time constant from a resource header; if the resource ID is missing or renamed in the `.rc` file, the dialog will fail to load at runtime.
- **Parent window coupling**: The optional `CWnd* pParent` parameter could introduce modal-modality bugs if the parent is destroyed while this dialog is active (typical for MFC modal dialogs on older Windows).
