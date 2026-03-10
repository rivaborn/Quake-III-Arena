# q3radiant/FindTextureDlg.h — Enhanced Analysis

## Architectural Role

This file defines a modal texture find-and-replace dialog for the Q3Radiant level editor. Q3Radiant sits outside the runtime engine (unlike `code/`); it's a standalone MFC-based authoring tool. This dialog bridges the editor's 3D viewport and scene representation: designers input search/replace shader names and invoke batch operations on selected brushfaces or the entire map. The singleton-like static methods (`setFindStr`, `setReplaceStr`, `isOpen`, `show`) suggest centralized lifecycle management—only one instance exists at a time, and external code can query or control it without direct instantiation.

## Key Cross-References

### Incoming (who depends on this file)
- `q3radiant/MainFrm.cpp` / `q3radiant/QE3.cpp` (menu/command dispatchers): likely create/show the dialog when user selects "Find Texture" from menu
- `q3radiant/Radiant.cpp` (main app): instantiates and manages dialog instance
- **No references in runtime engine** (`code/`): this is editor-only, never shipped in compiled game

### Outgoing (what this file depends on)
- MFC framework (`CDialog`, `CWnd`, `CString`, `DECLARE_MESSAGE_MAP`)
- Win32 dialog resource (implied `IDD_DIALOG_FINDREPLACE` resource ID in `.rc` file)
- The actual find/replace logic is in `.cpp` implementation file (not shown), likely calls texture/material system and brush/face selection APIs

## Design Patterns & Rationale

**Static Singleton Pattern**: Four static accessor methods (`setFindStr`, `setReplaceStr`, `isOpen`, `show`) manage a hidden global instance. This avoids passing dialog pointers through multiple call stacks and allows any subsystem (menu handler, keyboard shortcut, external tool) to trigger/check the dialog without coupling to instantiation details.

**MFC Dialog Data Binding**: `DoDataExchange(CDataExchange* pDX)` auto-marshals UI control values ↔ member variables (`m_strFind`, `m_strReplace`, `m_bSelectedOnly`, `m_bForce`, `m_bLive`). This is era-appropriate (1999–2005) desktop GUI practice; modern engines use immediate-mode imgui or data-driven scene graphs.

**Message Map Pattern**: `DECLARE_MESSAGE_MAP()` + message handlers (`OnBtnApply`, `OnOK`, `OnCancel`, `OnSetfocus*`) dispatch Windows events to methods. This is MFC boilerplate, not idiomatic to modern game engines.

## Data Flow Through This File

```
User Menu Selection ("Find Texture")
  → MainFrm::Show Dialog
    → CFindTextureDlg::show() [static, instantiates if needed]
      ↓
    Dialog Modal Loop (user types/selects options)
      ↓
    OnBtnApply() [or OnOK()]
      → Calls underlying find/replace implementation
        → Queries brush selection / full map
        → Regex/pattern match against texture names
        → Updates geometry / invalidates render cache
      ↓
    updateTextures(const char* p) [static]
      → Refreshes viewport/3D view with changes
```

The `m_bLive` flag likely triggers real-time preview; `m_bForce` may override constraints.

## Learning Notes

**Editor vs. Runtime Separation**: Radiant is entirely separate from the engine runtime. The architecture context shows no references from `code/` subsystems to `q3radiant/`. This clean boundary is key to Q3's design—the level editor was a commercial tool (sold separately), and the runtime engine ships without it. Modern engines often blur this (e.g., Unreal Editor uses engine code). Quake III kept them orthogonal.

**Pre-QVM UI Era**: This is hand-coded C++ dialog, not the script-driven UI system in `code/q3_ui` or `code/ui`. Those are QVM modules (bytecode) executed at runtime; Radiant's UI is pure native Win32/MFC. Designers never see Radiant UI in-game.

**CString and MFC Idioms**: The header uses `CString` (old COM-style string wrapper) rather than `std::string`. MFC predates STL adoption in Windows tools; this is characteristic of Visual C++ 6.0–era code (1999–2005).

## Potential Issues

- **Static state not thread-safe**: If any async operation (file I/O, texture loading) touches the dialog's members, race conditions are possible. MFC is single-threaded by design, so this is mitigated in practice, but fragile.
- **No error handling in header**: Success/failure of find/replace operations is not reflected in return types; success likely inferred by UI feedback or silent failure.
- **Singleton lifetime ambiguous**: When does the static instance get destroyed? Likely on app shutdown via `DestroyWindow()` override, but no explicit cleanup semantics in header.
