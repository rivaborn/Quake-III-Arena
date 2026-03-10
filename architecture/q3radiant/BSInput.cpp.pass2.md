# q3radiant/BSInput.cpp — Enhanced Analysis

## Architectural Role

BSInput is a lightweight UI component within the Radiant map editor (a tool-only module, not part of the runtime engine). It provides a generic parameterized dialog for collecting up to 5 float and 5 string values from the user during map authoring operations. Unlike the runtime game engine layers documented in the architecture, Radiant operates as a standalone Win32/MFC application with no connection to the QVM pipeline, network code, or game simulation—it is purely an offline content-creation tool.

## Key Cross-References

### Incoming (who depends on this file)
- Unknown without full Radiant codebase visibility; likely invoked by other dialog handlers (e.g., `CapDialog.cpp`, `DialogThick.cpp`, entity property panels) when generic multi-field input is needed
- No runtime engine dependencies (this is tooling, not shipped code)

### Outgoing (what this file depends on)
- Win32 MFC framework: `CDialog`, `CWnd`, `CDataExchange`, `DDX_Text`
- Standard Windows constants: `IDC_EDIT_FIELDn`, `IDC_STATIC_FIELDn`, `SW_HIDE`
- No dependencies on core engine subsystems (qcommon, renderer, game VM, etc.)

## Design Patterns & Rationale

**MFC Dialog Pattern (legacy Windows):**  
This code exemplifies late-1990s/early-2000s Windows UI development. `DoDataExchange()` implements two-way data binding between dialog controls and member variables—a pattern central to MFC that predates modern data-binding frameworks by decades.

**Dynamic Visibility Configuration:**  
`OnInitDialog()` hides field pairs (edit control + label) when the corresponding string label is empty, allowing a single dialog class to serve multiple use cases with variable field counts. This is a form of template instantiation—reusing the same dialog for 1–5 field scenarios without subclassing.

**Rationale for rigid 5-field limit:**  
Reflects the tool's pragmatic era: supporting enough field combinations for most editor tasks without implementing a dynamic form builder (which would have required more infrastructure).

## Data Flow Through This File

1. **Initialization:** Caller instantiates `CBSInput`, sets `m_strFieldN` labels to control visibility
2. **Display:** Dialog appears; `OnInitDialog()` hides unused field pairs based on empty string labels
3. **User Input:** Operator enters float/string values; MFC internally routes updates through `DoDataExchange()`
4. **Extraction:** Dialog is closed; caller reads `m_fFieldN` and `m_strFieldN` member variables to consume the input

## Learning Notes

**Obsolete Tooling Pattern:**  
This file encapsulates the Windows-centric, MFC-based editor that shipped with Q3A. Modern game editors (Unreal, Unity) use cross-platform frameworks (Qt, Gtk, C#/.NET, web-based) and implement dynamic form fields rather than hardcoded limits.

**Idiomatic to the Era:**  
The `//{{AFX_DATA_INIT}} ... //}}AFX_DATA_INIT` and `//{{AFX_DATA_MAP}} ... //}}AFX_DATA_MAP` pseudo-comments are MFC ClassWizard markers—a Visual Studio code-generation tool now obsolete. The entire handshake assumes a Visual Studio IDE and a pre-compiled MFC runtime.

**Contrast with Runtime Engine:**  
Unlike the runtime engine (which uses abstract VM syscalls and platform-independent file I/O), Radiant is tightly coupled to Win32 and MFC, making it non-portable. The architecture overview lists no Radiant dependencies from `code/`, confirming complete isolation.

## Potential Issues

- **Fixed field count:** Hardcoded 5-field limit inflexible for complex input scenarios
- **No validation:** Float fields accept any string; no range or format enforcement
- **Type limitations:** Only float + string; no enum selectors, file pickers, vector inputs, etc.
- **Win32-only:** No support for Linux/macOS level editing (bspc offline AAS compiler is cross-platform; the visual editor is not)
