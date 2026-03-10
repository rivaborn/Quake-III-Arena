# q3radiant/PatchDialog.cpp — Enhanced Analysis

## Architectural Role
This file implements the **Patch Inspector dialog** for the Q3Radiant level editor, providing interactive editing of Bézier surface patch control points. As a tool-only component, it bridges the editor's UI framework (MFC) with the underlying patch mesh infrastructure, offering real-time feedback to the 3D viewport through `Sys_UpdateWindows`. Unlike the runtime engine (which parses pre-compiled patches from BSP files), the editor allows artists to manually adjust individual control point positions and texture coordinates before export.

## Key Cross-References

### Incoming (who depends on this file)
- **q3radiant/MainFrm.cpp** (editor main window) — creates and shows the global `g_PatchDialog` instance via `DoPatchInspector()`
- **Editor selection/viewport** — calls `UpdatePatchInspector()` whenever the selection changes
- **q3radiant patch infrastructure** — `SinglePatchSelected()` queries the current patch from the editor's selection state

### Outgoing (what this file depends on)
- **q3radiant patch manipulation** — calls `Patch_NaturalizeSelected()`, `Patch_FitTexturing()`, `Patch_ResetTexturing()`, `Patch_SetTextureInfo()`
- **Editor viewport system** — `Sys_UpdateWindows(W_ALL, W_CAMERA)` to refresh the 3D view after edits
- **Editor registry persistence** — `SaveRegistryInfo()` / `LoadRegistryInfo()` to preserve window position across sessions
- **q3radiant selection system** — `SinglePatchSelected()` to retrieve the currently selected patch object
- **MFC framework** — inherits from `CDialog`, uses `DDX_*` data exchange, message map dispatch

## Design Patterns & Rationale

1. **MFC Dialog Pattern**  
   - Inherits from `CDialog` and implements standard MFC message handling (`BEGIN_MESSAGE_MAP`, `ON_*` macros)
   - `DoDataExchange()` binds dialog controls to member variables (`m_fX`, `m_fZ`, etc.) — a declarative two-way binding approach
   - Why: MFC was the standard UI framework for Q3Radiant on Windows in the late 1990s; this pattern allows controls to auto-sync without manual getter/setter code

2. **Modeless Dialog Lifecycle**  
   - `DoPatchInspector()` creates the dialog once and keeps it alive (` GetSafeHwnd() == NULL` check prevents re-creation)
   - Window rect persisted via registry (`LoadRegistryInfo` / `SaveRegistryInfo` on `OnDestroy()`)
   - Why: Artists frequently toggle the inspector on/off; modeless allows the main editor to remain responsive

3. **Spinner Control Pattern**  
   - Up-down spinners paired with text edit fields for continuous-value adjustment
   - `UpdateSpinners()` interprets `iDelta` sign to apply relative transformations (e.g., `1 - scale` vs. `1 + scale`)
   - Why: Avoids keyboard input errors for fine-tuning texture alignment parameters across the selected patch

4. **Data Binding via Row/Column Selection**  
   - `m_wndRows` / `m_wndCols` combo boxes select which control point to display/edit
   - `UpdateRowColInfo()` reads from `m_Patch->ctrl[c][r]` and pushes to member variables via `UpdateData(FALSE)`
   - `OnApply()` writes back via `UpdateData(TRUE)` and direct struct assignment
   - Why: Allows per-control-point editing without reloading the entire mesh; matches patch topology

## Data Flow Through This File

1. **Initialization** (`DoPatchInspector` → `OnInitDialog`):
   - Dialog created with modeless flag
   - Window position restored from registry
   - Spinner ranges set (0–1000 for normalized values)

2. **Selection Update** (`UpdatePatchInspector` → `GetPatchInfo`):
   - Retrieve currently selected patch via `SinglePatchSelected()`
   - Populate row/column combo boxes from `m_Patch->height` / `m_Patch->width`
   - Call `UpdateRowColInfo()` to display control point at [0][0]

3. **User Edits Patch Position/UV**:
   - User changes combo selection → `OnSelchangeComboRow` / `OnSelchangeComboCol` → `UpdateRowColInfo()`
   - Reads control point position (XYZ) and texture coords (ST) from `m_Patch->ctrl[c][r]`
   - User modifies text fields
   - `OnApply()` writes back and sets `m_Patch->bDirty = true`
   - `Sys_UpdateWindows(W_ALL)` refreshes all viewports

4. **User Adjusts Texture Transform** (spinners):
   - Spinner delta → `UpdateSpinners()`
   - Builds `texdef_t` with scale/shift/rotate
   - Calls `Patch_SetTextureInfo()` to apply transformation across entire patch
   - `Sys_UpdateWindows(W_CAMERA)` updates preview

5. **Cleanup** (`OnDestroy`):
   - Window rect saved to registry for next session

## Learning Notes

- **Patch Representation**: The editor stores patches as a 2D grid of control points (`ctrl[width][height]`), each holding position (XYZ) and texture coordinate (ST). This maps directly to Quake III's Bézier surface format, which the runtime engine interpolates and renders in `code/renderer/tr_curve.c`.

- **Texture Coordinate Manipulation**: Unlike most meshes, Q3 patches have an explicit UV editor. This is critical because Bézier surfaces can distort textures if control points are not carefully placed. The "fit," "naturalize," and "reset" functions are shortcuts for common texture layout problems.

- **No Runtime Equivalent**: The game engine (cgame, game VMs, renderer) does not have an interactive patch editor — patches are read-only from the BSP. This is entirely a level-authoring tool.

- **MFC as Transitional Technology**: By the time Q3 shipped (2000), MFC was already aging. Modern engines use custom immediate-mode UIs or Qt. The modeless dialog pattern here is idiomatic to 1990s Windows game tools; it pre-dates scene graphs and property panels.

- **Bounded Array Access**: The bounds checks (`r >= 0 && r < m_Patch->height`) protect against out-of-bounds combo selection, but assume `m_Patch->height` and `m_Patch->width` are always valid. If a patch object is corrupted or half-initialized, this could still crash.

## Potential Issues

1. **Unimplemented SetPatchInfo()** — Function exists but is empty (line ~208), suggesting incomplete refactoring or a planned feature that never shipped.

2. **Empty OnSelchangeComboType** Handler (line ~138) — TODO comment indicates this combo box has no handler; users cannot actually change patch type from the dialog.

3. **No Null-Safety on m_Patch** — `UpdateRowColInfo()` and `OnApply()` check bounds but do not null-check `m_Patch` before dereferencing `->ctrl[c][r]`. If `m_Patch` is null and bounds pass, a segfault occurs. (Though this is partially mitigated by the `if (m_Patch != NULL)` guard, it's still defensible in production code.)

4. **Registry Persistence Assumes Valid Path** — `LoadRegistryInfo()` / `SaveRegistryInfo()` calls assume the registry key exists; on a clean Windows install or non-Windows systems, these may silently fail, causing window position to reset.
