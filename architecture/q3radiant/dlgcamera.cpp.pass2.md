# q3radiant/dlgcamera.cpp — Enhanced Analysis

## Architectural Role

This file implements the **Camera Inspector dialog**, a specialized tool within the Radiant level editor for authoring cinematic camera paths via spline animation. It bridges the editor's MFC UI layer with the spline math system (`code/splines/`), managing the relationship between user interaction (dialog controls), persistent spline data (`g_splineList`), and real-time viewport preview. The file is strictly an **editor tool**—it has zero presence in the runtime engine.

## Key Cross-References

### Incoming (who depends on this file)
- **Radiant editor main loop** (`q3radiant/MainFrm.cpp`, `q3radiant/QE3.cpp`): calls `showCameraInspector()` to spawn the dialog
- **Editor selection system** (`g_qeglobals`): reads/writes `d_select_mode` (sel_editpoint, sel_addpoint) and `selectObject` to integrate spline editing into the main 3D viewport
- **Registry/preferences system** (`LoadRegistryInfo`, `SaveRegistryInfo`): persists dialog window position across sessions

### Outgoing (what this file depends on)
- **Global spline list** (`g_splineList`): encapsulates all camera path data; this dialog is the primary UI client for its state mutations (setName, setBaseTime, addEvent, addTarget, buildCamera, etc.)
- **Spline math library** (`code/splines/splines.h`): supplies `idCameraEvent`, `idCameraPosition`, and spline evaluation for real-time camera queries
- **Editor viewport** (`g_pParentWnd->GetCamera()`): pushes computed camera position/angles per-frame during animation scrub
- **Editor UI subsystem** (`Sys_UpdateWindows`): triggers viewport refresh after state changes
- **Dialog/event infrastructure** (`CDlgEvent`, `CNameDlg`, `CCameraTargetDlg`): spawns child dialogs for event/target/file operations
- **Windows MFC** (`CDialog`, `CWnd`, message maps): platform-specific UI framework

## Design Patterns & Rationale

**MFC Two-Way Data Binding:**  
`DoDataExchange()` syncs UI controls ↔ member variables (m_strName, m_fSeconds, m_numSegments). This is idiomatic pre-.NET UI programming; modern engines use data-binding frameworks (e.g., property observables). The pattern isolates control state into C++ objects.

**Global Singleton State:**  
`g_dlgCamera` and `g_splineList` are editor-wide globals. This avoids argument threading but couples the dialog tightly to global state—reflects Radiant's monolithic architecture.

**Message-Based Event Dispatch:**  
Windows `BEGIN_MESSAGE_MAP()` routes UI events (button clicks, scrollbar moves) to handler functions. This is standard MFC; modern engines use event buses or callbacks.

**Registry Persistence:**  
Dialog position saved/restored via Windows registry (`LoadRegistryInfo`/`SaveRegistryInfo`). Reflects early-2000s Windows UI practices; modern tools use JSON/TOML config files.

**Dual-Mode Selection:**  
`m_editPoints` radio button toggles `g_qeglobals.d_select_mode` between `sel_editpoint` (modify path) and `sel_addpoint` (add new control points). This integrates spline editing into the main viewport selection system rather than modal-only dialogs.

## Data Flow Through This File

```
User Input
  ↓
MFC Message Handler (e.g., OnHScroll)
  ↓
Query/Update g_splineList State
  ↓
Compute Camera Position (g_splineList->getCameraInfo)
  ↓
Push to g_pParentWnd->GetCamera() (viewport preview)
  ↓
Sys_UpdateWindows(W_XY | W_CAMERA) (refresh views)
```

**Key state transformations:**
- **Scrollbar position (0–max)** → normalized time (0–1) → milliseconds → spline parameter space → (origin, dir, FOV)
- **Spline edits (name, time, targets, events)** → `buildCamera()` → recomputed animation curves → updated scroll range

**Animation scrubbing flow** (`OnHScroll`):
1. Normalizes scroll position to [0, getTotalTime()]
2. Converts to milliseconds (multiply by 1000)
3. Calls `getCameraInfo(p, &origin, &dir, &fov)` to evaluate spline at time *p*
4. Extracts Euler angles (yaw via `atan2`, pitch via `asin`) and pushes to viewport camera
5. The 4.0 scroll multiplier implements fine-grained control (quarters of a second)

## Learning Notes

**Idiomatic to this era & engine:**
- **MFC dialogs for specialized tools:** Radiant uses modeless dialogs (stays open) for persistent editor tools, unlike older modal patterns. This allows simultaneous viewport interaction.
- **Global editor state** (`g_qeglobals`): Radiant centralizes editor flags and selection state here; modern engines prefer entity-component systems or property bags.
- **Viewport camera as mutable state**: The dialog directly manipulates `g_pParentWnd->GetCamera()->Camera()`, pushing camera origin/angles per-frame. Modern engines would queue this via a command/event system.
- **Spline as "object"**: `g_splineList` appears to be a wrapper around a single spline; no object pooling or named asset system (unusual by modern standards).

**Connections to engine concepts:**
- **Interpolation/Extrapolation**: Spline animation is client-side (like the cgame VM's particle interpolation), not server-authoritative. Useful for cinematics/replays.
- **Entity Event System**: Similar to cgame VM's `EV_*` events—animations trigger discrete events (OnBtnAddevent) that can fire gameplay actions.
- **Registry Persistence**: Parallels the engine's `Cvar_WriteVariables` for config; editor preferences use the same Windows registry as server cvars.

**What modern engines do differently:**
- Asset serialization (JSON/ASDL/msgpack) instead of ad-hoc `.camera` file formats
- Event buses (`CAM_PathCreated`, `CAM_TimeChanged`) instead of direct global mutations
- Property panels (inspector pattern) vs. separate floating dialogs
- Undo/redo system (not present here) via command objects
- Constraint solvers (quaternion slerp, Catmull–Rom) vs. raw spline queries

## Potential Issues

1. **Incomplete Implementation**: Eight methods are TODO stubs (`OnTestcamera`, `OnBtnDeletepoints`, `OnDblclkComboSplines`, etc.), suggesting the dialog was partially completed or abandoned.

2. **Magic Numbers**: Scroll range multiplier (4.0), time scaling (1000), hardcoded refresh flags—should be named constants or config.

3. **va() Varargs Buffer**: The static `va()` function uses a 32 KB buffer shared by nested calls. Safe here due to shallow call stacks, but inherently unsafe if string usage grows.

4. **No Input Validation**: File load/save dialogs don't check if files exist or handle parse errors gracefully.

5. **Global Coupling**: Direct manipulation of `g_splineList` and `g_qeglobals` makes unit testing and refactoring difficult. No layer of indirection (e.g., ISplineManager interface).

6. **Precision Loss in UI**: Scroll position stored as `int`; real-time animation scrub may jitter if frame time doesn't align perfectly with scroll quantization.
