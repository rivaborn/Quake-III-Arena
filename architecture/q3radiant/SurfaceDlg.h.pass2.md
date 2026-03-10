# q3radiant/SurfaceDlg.h — Enhanced Analysis

## Architectural Role
`SurfaceDlg.h` defines a modeless/modal MFC dialog for editing texture and patch mesh properties within the Q3Radiant level editor. This is a **development-time tool**, not part of the runtime engine—it bridges user interactions with the underlying brush/patch geometry management system during map authoring. The dialog encapsulates the UI controls and logic for texture transformations (shift, rotation, scale) and patch mesh parameters (width, height).

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main frame** (`MainFrm.cpp`): Likely instantiates `CSurfaceDlg` when the user selects "Surface Properties" or similar menu item
- **Entity/brush selection system**: When the user selects a face or patch, the dialog state is populated via `GrabPatchMods()` or similar
- **Radiant undo/redo system**: Changes made through this dialog may trigger undo/redo snapshots

### Outgoing (what this file depends on)
- **Brush/patch management layer** (e.g., `Brush.cpp`, `PMESH.cpp`): Likely calls methods on the current selection to apply texture mods via `SetTexMods()`
- **Map state** (`Map.cpp` or similar): Reads/writes surface properties from the currently selected entities
- **Undo/redo infrastructure**: Likely wraps texture modifications in undo transactions
- **OpenGL/3D viewport**: Probably triggers viewport redraw after applying changes

## Design Patterns & Rationale

1. **MFC Dialog Pattern**: Inherits from `CDialog`, uses `DoDataExchange` for data binding and validation—standard Windows GUI architecture of 2000s-era C++ tools
2. **Spin Button Controls**: Seven `CSpinButtonCtrl` members handle numeric input for shift, scale, rotation, width, height—common pattern for constrained numeric UI in level editors
3. **Modeless vs. Modal**: The presence of `OnApply()` (distinct from `OnOK()`) suggests a **modeless dialog** that applies changes in real-time without closing
4. **State Encapsulation**: Member variables (`m_shift[]`, `m_rotate`, `m_scale[]`) cache patch/surface modification state locally before commit
5. **Method Symmetry**: `SetTexMods()` / `GetTexMods()` pair mirrors bidirectional sync between UI and underlying geometry

## Data Flow Through This File

1. **Read Phase** (`GrabPatchMods()` called when patch/face selected):
   - Query current patch dimensions and texture transform from `Map.cpp` / brush selection
   - Populate `m_shift[]`, `m_rotate`, `m_scale[]` member vars
   - Bind to spinner controls via `DoDataExchange(FALSE)`  ← display in UI

2. **Interaction Phase** (user adjusts spinners):
   - Spinner messages (`OnDeltaPosSpin`, `OnHScroll`, `OnVScroll`) update member vars
   - Optionally trigger viewport refresh to preview changes

3. **Commit Phase** (`SetTexMods()` called when user clicks Apply/OK):
   - `DoDataExchange(TRUE)` collects UI values into member vars
   - Apply modifications to currently selected brushes/patches
   - Trigger viewport redraw and possibly undo checkpoint

## Learning Notes

**What this file teaches about Quake III tooling:**
- Q3Radiant is a **Windows-only MFC application** (evident from `#pragma once`, `CREATESTRUCT`, `CWnd`, etc.)—porting to modern platforms would require replacing MFC with Qt, wxWidgets, or similar
- **Texture transformation UI** is separate from the raw BSP export: the dialog manages **visual editing** on the in-editor representation, not direct BSP data
- **Spinner controls** were the standard for numeric input in early-2000s game editors (no slider ranges visible here, but spin controls limit precision)
- **Patch mesh density** (width/height) is mutable post-creation, unlike modern engines where mesh topology is typically locked

**Idiomatic patterns of this era:**
- No async undo/redo; changes likely queued synchronously
- No viewport-independent property panel; dialog is modal/blocking
- Integer-only patch dimensions (`int m_nHeight`, `int m_nWidth`)—floats for transforms suggest early patch deformation work predated full Bézier support

**Modern equivalent:**
In contemporary level editors (Unreal, Unity), surface property editing would be:
- Part of a persistent **Inspector panel** rather than a modal dialog
- Support **undo/redo per keystroke** rather than on commit
- Use **sliders + number fields** together for range feedback
- Expose via **context menu** on selected faces, not a top-level menu

## Potential Issues

- **Integer patch dimensions** (`m_nHeight`, `m_nWidth`) may lose precision if patches need sub-texel resolution adjustments
- **No validation visible** in the header—if `SetTexMods()` applies unbounded transforms, invalid texture coordinates could crash the renderer during playtest
- **Thread safety**: If applied in a modeless dialog while another thread accesses the selection, concurrent modification could corrupt brush geometry
