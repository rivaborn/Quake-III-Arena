# q3radiant/RotateDlg.h — Enhanced Analysis

## Architectural Role

`CRotateDlg` is a modeless dialog in the Radiant level editor's UI layer that enables precise angular transformation of brush and entity selections. It bridges user input (numeric fields and spinner controls) to the core level geometry manipulation pipeline (`Select.cpp`, `Brush.cpp`), allowing rotation by named axes (X, Y, Z) with both direct entry and incremental adjustment via spin buttons. This is one of several transformation dialogs (alongside scale, position) that form the editor's non-destructive property adjustment interface.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant/MainFrm.cpp** (inferred): Opens/closes dialog via menu or toolbar; routes apply commands
- **q3radiant/Select.cpp** (inferred): Receives rotation parameters and applies them to selected brush faces/entities
- **q3radiant/Brush.cpp** (inferred): Performs actual geometric rotation transformations on brushes
- **q3radiant/WIN_XY.cpp, WIN_Z.cpp** (inferred): May trigger rotation dialog from viewport context menus

### Outgoing (what this file depends on)
- **Windows MFC framework**: `CDialog`, `CSpinButtonCtrl`, `CString`, `DoDataExchange`, message map macros
- **q3radiant's selection/brush system** (via ApplyNoPaint): Applies rotation deltas to selected geometry without full redraw
- **Radiant's undo/redo system** (inferred): Likely integrated via cmdlib or undo manager

## Design Patterns & Rationale

**MFC Dialog Pattern**: Classic Windows modal/modeless dialog using resource templates (`IDD_ROTATE`). The `CSpinButtonCtrl` paired with `CString` fields implements a common "spinner + text box" pattern: users can either click/drag the spinner for incremental changes or type exact values.

**DDX/DDV (Data Exchange/Validation)**: `DoDataExchange` synchronizes UI controls with C++ member variables (`m_strX/Y/Z`). This abstraction decouples validation logic from message handlers.

**ApplyNoPaint() Optimization**: Suggests the editor distinguishes between "interactive" updates (no repaint cost during rapid spinner clicks) and "final" commits (full geometric rebuild). This is idiomatic to 1990s/early-2000s editors where screen redraws were expensive.

**Spin Button Message Routing**: `OnDeltaposSpin1/2/3` handlers respond to each axis independently, allowing per-axis incremental adjustment without requiring dialog re-entry.

## Data Flow Through This File

1. **Inbound**: User selects brush(es)/entity in viewport → chooses "Rotate" from menu → dialog spawns with current rotation or zeros
2. **User Input**: 
   - Direct: Types values into m_strX/Y/Z, clicks OK/Apply
   - Incremental: Clicks spinner buttons (OnDeltaposSpin1/2/3), each click adjusts string value
3. **Outbound**: `OnOK()` or `OnApply()` → calls `ApplyNoPaint()` → updates selection geometry in viewport
4. **State**: Dialog retains values across reuse (typical for modeless dialogs); user may click Apply multiple times before OK

## Learning Notes

**Idiomatic to Q3 Era**:
- Heavy reliance on MFC resource macros (`AFX_DATA`, `AFX_VIRTUAL`, `AFX_MSG`) for code generation
- No separation between UI logic and domain logic (dialog directly manipulates geometry)
- Resource-based UI definition rather than programmatic UI construction

**Modern Engines Differ**:
- Would likely use a dedicated property inspector panel rather than modal dialogs
- Spin controls might be replaced with drag-to-adjust sliders or curves
- Property changes would immediately preview without explicit Apply button

**Editor Usability Insight**: The three separate spin controls for X/Y/Z suggest the editor supports constrained rotation (rotate around single axis), which is fundamental to 3D modeling workflows.

## Potential Issues

- **No validation hint**: `DoDataExchange` sets `IDD = IDD_ROTATE` but actual DDV rules are in the implementation (.cpp), making it hard to verify safe input ranges without reading the source
- **ApplyNoPaint() name**: Unclear whether this is a commit or a preview; modern feedback would make this explicit

---
