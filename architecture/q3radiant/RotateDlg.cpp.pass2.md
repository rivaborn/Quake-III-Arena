# q3radiant/RotateDlg.cpp — Enhanced Analysis

## Architectural Role

This file implements a modeless rotation dialog for the Quake III level editor (q3radiant). It bridges user input (text fields and spinner controls) to the selection system's rotation pipeline. As a UI component of an offline map-authoring tool, it sits outside the runtime engine entirely — the cross-reference context provided focuses on runtime subsystems (botlib, game VM, renderer), whereas this dialog is part of the editor's persistent window suite.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant main window** likely spawns `CRotateDlg` modally or modeless during editor initialization or menu activation
- The dialog's life cycle is managed by MFC framework (`CDialog` base class handling `OnInitDialog`, `OnOK`, etc.)
- Radiant's document or view layer likely registers/unregisters this dialog when selection state changes

### Outgoing (what this file depends on)
- **`Select_RotateAxis(int axis, float angle)`** — defined elsewhere in q3radiant (likely `select.c` or `SELECT.CPP`), implements the core rotation operation on selected brushes/entities
- **MFC framework** (`CWnd`, `CDialog`, `DDX_*` data exchange macros, spinner control messaging)
- **No engine dependencies** — this is purely an editor tool with zero runtime relevance

## Design Patterns & Rationale

**MFC Dialog Pattern (1990s Microsoft C++ UI Framework):**
- `DoDataExchange()` implements two-way data binding: `UpdateData(TRUE)` pulls from controls into member variables; `UpdateData(FALSE)` pushes member variables back to controls
- `ON_BN_CLICKED` and `ON_NOTIFY` macros define the message map — routing button clicks and spinner notifications to handler functions
- Spinner controls (`CSpinButtonCtrl`) wrap the three axis input fields, allowing click-to-increment or direct text entry

**Why this structure?**
- The 0–359° range on spinners prevents invalid angle input at the UI level
- Three separate `Select_RotateAxis` calls (one per axis) suggest the underlying selection system applies rotations sequentially per-axis, not as a composed 3D matrix transformation
- `OnApply()` is called by both `OnOK()` (dialog close + apply) and spinner delta handlers (live preview), allowing partial or incremental rotations

## Data Flow Through This File

1. **User Input**: Type or spin a rotation value for X/Y/Z axis → control focus lost or spinner clicked
2. **Parse**: `UpdateData(TRUE)` converts text fields to member vars; `atof()` parses string → float
3. **Gate**: Non-zero check avoids spurious 0° rotations
4. **Apply**: `Select_RotateAxis(axis_index, angle_float)` dispatches to the selection system
5. **Feedback**: Spinner delta handlers fire immediately; user sees live preview (if underlying system supports it); text fields remain unsynchronized with the preview (one-way flow on apply)

## Learning Notes

- **MFC Era (1999)**: This code predates modern C++ UI frameworks (Qt, wxWidgets) and demonstrates the Windows-only, message-driven paradigm of the era. The `AFX_` macros and `//{{AFX_...}}` comments are MFC's code generator boundaries (wizard-generated code).
- **Modeless Dialog**: Unlike modal dialogs that block user interaction, this dialog likely coexists with the 3D viewport, enabling iterative refinement. The empty `ApplyNoPaint()` stub suggests an intent to separate logic updates from screen refresh (see issue below).
- **No undo/redo visible**: The dialog applies rotations immediately without pushing undo history. This is likely handled by `Select_RotateAxis` itself or the parent selection system.
- **Angle representation**: Euler angles (X/Y/Z) are used rather than axis-angle or quaternions, typical for level editors (intuitive, matches Quake's pitch/yaw/roll convention).

## Potential Issues

1. **Orphaned `ApplyNoPaint()` method**: Declared but never called; empty body suggests incomplete refactoring (perhaps intended to defer screen updates during batch rotations, but never wired up). **Action**: Remove or complete.
2. **No float validation**: `atof()` silently converts malformed strings to 0.0. Edge cases like "abc" or "1.5.6" pass through. Input sanitization should occur before or after `atof()`.
3. **Spinner range ≠ practical range**: Spinners bound to 0–359°, but rotation angles can exceed 360° (multiple full rotations). Entering "450" in text field will apply; spinner will wrap to 90.
4. **No inter-axis coupling**: Three independent spinners imply rotations are applied in sequence per-axis, not as a single composed transformation. If the underlying `Select_RotateAxis` has order-dependent behavior, the dialog provides no hint to the user.

---

**Generated as second-pass analysis incorporating architecture subsystem context and cross-references.**
