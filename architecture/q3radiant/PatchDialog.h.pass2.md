# q3radiant/PatchDialog.h — Enhanced Analysis

## Architectural Role

`PatchDialog.h` implements the UI control panel for editing individual Bézier patch mesh properties within Q3Radiant (the level editor). It serves as the **parameter mutation layer** between the editor's spatial viewport and the runtime patch representation: converting user slider/spinner input (scale, shift, rotation, UVs) into `patchMesh_t` updates. This dialog bridges the editor tool (MFC/Win32 UI) and the engine's Bézier patch subsystem, which appears throughout the rendering pipeline (`code/renderer/tr_curve.c`) and BSP compiler (`q3map/patch.c`).

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main frame** (`MainFrm.*`, `RadiantView.*`) — creates/shows/hides the dialog when a patch surface is selected
- **Selection/brush management** (`SELECT.cpp`, `Brush.cpp`) — likely passes the selected `patchMesh_t*` to `CPatchDialog::SetPatchInfo()`
- **Entity/surface plugin infrastructure** — may instantiate the dialog for patch editing operations

### Outgoing (what this file depends on)
- **`patchMesh_t` definition** — from the engine's shared game/editor headers (likely `code/game/bg_public.h` or `q3radiant/QERTYPES.H` adapter)
- **Renderer curve system** (`code/renderer/tr_curve.c`) — provides the semantic understanding of patch density, subdivision, and Bézier control points that this dialog's spinners/combos map to
- **Patch tessellation parameters** — row/column count combos control the patch's evaluated grid density (visible in `tr_curve.c: R_SubdividePatchToGrid`)
- **Texture scaling/shifting** — the float fields (`m_fHScale`, `m_fHShift`, etc.) correspond to per-patch surface properties applied during shader binding

## Design Patterns & Rationale

**MFC Dialog Data Exchange (DDX) Pattern:**
- The `//{{AFX_DATA}}` markers and `DoDataExchange()` virtual override are MFC's compile-time code-generation hooks
- Auto-wiring of control handles to member variables (`m_wndVShift`, `m_fZ`) via the ClassWizard tool
- This was idiomatic for late-1990s/early-2000s Windows dialogs before WinForms/XAML

**Dual Notification Pipeline:**
- `UpdateInfo()` → display current `patchMesh_t` state in UI controls
- `SetPatchInfo()` / `GetPatchInfo()` → marshal data to/from the underlying patch mesh
- `UpdateSpinners(bool bUp, int nID)` → handle incremental adjustments (arrows on spinner controls)
- This separation allows the dialog to remain stateless and reusable across multiple patch selections

**Why spin buttons + combo boxes:**
- Continuous numeric adjustment (rotation, scale, shift) via spinner feedback
- Discrete enumeration (rows/cols/type) via dropdown (reflects compile-time patch grid options)
- Matches Quake III's design philosophy: artist-friendly discrete subdivision levels rather than arbitrary tessellation

## Data Flow Through This File

1. **Selection → Dialog Activation:**
   - Editor user clicks a patch surface in viewport
   - Brush/entity selection system identifies `patchMesh_t*`
   - Dialog is created/shown; `CPatchDialog(patchMesh_t*)` constructor called

2. **State Display:**
   - `OnInitDialog()` → populates combo boxes (valid row/col/type options)
   - `SetPatchInfo()` → extracts current `m_Patch` properties into UI fields (`m_fRotate`, `m_fHScale`, etc.)
   - `UpdateInfo()` / `UpdateRowColInfo()` → re-syncs dependent UI state when user changes row/col selection

3. **User Edits:**
   - Spinner clicks, combo selections, text field input trigger `OnDeltaposSpin()`, `OnSelchangeCombo*()`, and `DoDataExchange()`
   - MFC validation/sync happens automatically via `UpdateData(TRUE)` in message handlers

4. **Submission:**
   - `OnOK()` → calls `GetPatchInfo()` to marshal UI state back into `m_Patch`
   - `OnApply()` → non-modal commit (apply changes without closing)
   - Editor viewport is notified and re-renders the patch with new parameters

## Learning Notes

**Quake III Patch Architecture Insights:**
- Patches are **not procedural** — they have fixed row/column counts (typically 3, 5, 9, etc.) set at authoring time
- The renderer tessellates them (`R_SubdividePatchToGrid`, `R_Tess*` flow in `tr_curve.c`) into triangle meshes each frame (expensive but necessary for LOD/clipping)
- Texture coordinate scaling/shifting is per-patch metadata, allowing artists to tweak UVs without re-modeling

**Era-Specific Patterns (late 1990s):**
- MFC dialogs were the industry standard for Win32 tools before C#/WPF
- ClassWizard's code generation and `{{` markers are MFC boilerplate; modern editors would use data-binding or reactive frameworks
- No visible unit tests, error handling is implicit (MFC exceptions), validation happens in message handlers

**Connection to Engine Subsystems:**
- This dialog is **not** part of the runtime engine; it's a pure editing tool
- Changes only take effect when the map is saved and recompiled through `q3map` (which processes patches via `code/bspc/` and `q3map/patch.c`)
- The dialog's UI parameters must align with `patchMesh_t` struct layout and the renderer's `patchHeight_t` grid evaluation

## Potential Issues

- **No obvious bounds checking** on float fields (`m_fRotate`, `m_fHScale`) — could accept invalid values (e.g., negative scales, unbounded rotation)
- **Combo box population** (valid rows/cols) is hard-coded; if the renderer supports new grid sizes, this dialog must be updated separately
- **Missing async validation** — no preview or real-time update while typing (only on control focus loss)
- **Tight coupling to patchMesh_t layout** — if the engine patch struct changes, this dialog's member fields and `GetPatchInfo()`/`SetPatchInfo()` must be manually resynchronized
