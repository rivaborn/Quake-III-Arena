# q3radiant/PatchDensityDlg.cpp — Enhanced Analysis

## Architectural Role

This file implements a lightweight MFC dialog for the Radiant level editor that mediates user selection of Bezier patch tessellation density. It bridges the UI layer (combo box selection) to the mesh generation subsystem via `Patch_GenericMesh()`, parameterizing patch complexity without exposing raw numeric ranges to the user. It's a pure editor-tool component with no runtime engine role.

## Key Cross-References

### Incoming (who depends on this file)
- **Radiant main window** (`q3radiant/MainFrm.h`) likely invokes this dialog via a menu or toolbar command when the user requests patch density adjustment
- **Dialog framework** (MFC/Windows) dispatches `OnOK()` and `OnInitDialog()` callbacks

### Outgoing (what this file depends on)
- **`Patch_GenericMesh()`** — generates a new patch mesh; implementation not shown in cross-ref but called with (width, height, viewType) parameters
- **`g_pParentWnd->ActiveXY()->GetViewType()`** — global editor state accessor chain retrieving the current orthogonal view orientation (XY/XZ/YZ)
- **`Sys_UpdateWindows(W_ALL)`** — editor-level screen refresh function (not the engine `Sys_*` layer)
- **`CDialog`, `CWnd`, `DDX_Control`** — MFC framework base classes for dialog lifecycle and data binding

## Design Patterns & Rationale

**Parameterization lookup table:** The hardcoded `g_nXLat[]` array (`{3,5,7,9,11,13,15}`) maps combo box indices to patch control-point densities. This abstraction separates UI presentation (small integers 0–6) from geometric meaning (odd patch dimensions). The selection of odd numbers ensures a centered control point and symmetric tessellation.

**MFC data exchange:** `DoDataExchange()` with `DDX_Control()` auto-binds UI widgets to member variables (`m_wndWidth`, `m_wndHeight`), following MFC idiom for decoupling layout changes from code logic. The empty `AFX_DATA_INIT` and `AFX_DATA_MAP` comment blocks indicate this dialog only manages widget state, not data fields.

**Validation gating:** The `OnOK()` range check (`>= 0 && <= 6`) prevents out-of-bounds lookups; this is defensive but tight coupling suggests the combo box is already constrained by MFC at initialization.

## Data Flow Through This File

1. **User selects densities** → combo boxes update internal `m_wndWidth` / `m_wndHeight` selection indices
2. **User clicks OK** → `OnOK()` retrieves indices via `GetCurSel()` and bounds-checks them
3. **Table lookup** → indices map to control-point densities via `g_nXLat[]`
4. **Mesh generation** → `Patch_GenericMesh(width, height, viewType)` creates patch with specified tessellation in the active viewport
5. **Screen update** → `Sys_UpdateWindows(W_ALL)` redraws editor viewports to display the new patch

## Learning Notes

**Editor-specific conventions:** This file uses the global `g_pParentWnd` accessor pattern, which is idiomatic to mid-2000s editor architecture before widespread MVC/MVVM adoption. Modern editors typically inject the window context or use event buses.

**Odd control-point densities:** The Bezier patch spline convention in Q3A (inherited from older engines) uses odd-numbered control point grids. Understanding why (symmetric tessellation, centered weights) is fundamental to patch-based level design.

**MFC data binding:** The `DoDataExchange()` pattern, now obsolete, represents pre-STL/pre-template UI frameworks. Windows/platform code of this era heavily relied on RTTI and message maps rather than callbacks or signals.

**Dialog modal flow:** The dialog blocks user interaction until dismissed (modal `CDialog`), typical of early-2000s editors. Modern non-modal or floating palette UI is preferable for iterative design workflows.

## Potential Issues

**No undo/redo integration:** Calling `Patch_GenericMesh()` directly from `OnOK()` likely bypasses the editor's undo system. If the editor implements undo, this action may not be reversible.

**Redundant validation:** The range check is defensive but suggests the combo box population (`SetCurSel(0)` in `OnInitDialog()`) is not strongly typed; a dropdown with fixed options should be intrinsically valid.

**Hard-coded lookup table:** Future patch density options require code edit + recompile. A data-driven table (loaded from config) would be more maintainable for tools.
