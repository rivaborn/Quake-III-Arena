# q3radiant/PrefsDlg.cpp — Enhanced Analysis

## Architectural Role

PrefsDlg.cpp implements the preferences dialog for the Q3Radiant level editor — a standalone Windows MFC application that **does not ship with the runtime engine** and has no bearing on client/server/game logic. It manages editor-centric configuration (UI layout, autosave intervals, file paths to external tools like q3map and the game executables, rendering toggles like shader/GL lighting) persisted to the Windows registry. This file bridges editor UI state to the broader radiant codebase through global callbacks like `Sys_UpdateWindows()` and `Undo_SetMaxSize()`.

## Key Cross-References

### Incoming (who depends on this file)
- **MainFrm.cpp** (main editor window): likely instantiates and shows this dialog in response to menu commands
- **Radiant application object** (`Radiant.cpp`): uses `AfxGetApp()->GetProfile*` and `WriteProfile*` to delegate registry I/O
- **Global functions**: `Sys_UpdateWindows()` called in `OnOK()` to refresh all editor viewports after preference save
- **Undo subsystem**: `Undo_SetMaxSize(m_nUndoLevels)` configures max undo stack depth

### Outgoing (what this file depends on)
- **Windows registry** (via MFC `AfxGetApp()->GetProfileInt/String`): sole persistence backend; no disk files
- **Global state**: reads `g_strAppPath` and `g_pParentWnd`; writes editor viewport state indirectly via `Sys_UpdateWindows()`
- **MFC framework**: CDialog, CFileDialog (Windows file picker), spin/slider/combo controls
- **Editor globals**: grid status, window refresh commands
- **External tool paths**: q3map compiler, game executables (Quake2, Quake3), PAK files — user-configurable per preference

## Design Patterns & Rationale

- **MFC Dialog + DDX pattern**: Automatic marshalling of UI control state ↔ member variables via `DDX_Check`, `DDX_Text`, `DDV_MinMaxInt` eliminates manual widget polling/update boilerplate. Standard for 1990s Windows C++ UIs.
- **Registry-backed preferences**: `PREF_SECTION` and `INTERNAL_SECTION` keys allow per-user settings survival across restarts; no file locking or merge complexity (acceptable for single-user desktop tool).
- **Game selection conditional UI**: Lines 261–275 enable/disable PAK file and internal BSP options based on `m_strWhatGame` (`Quake3` disables Q2-specific paths). Reflects editor's multi-game heritage (originally shared Q2/Q3 toolchain).
- **Dual-initialization pattern** (constructor + `LoadPrefs()`): Constructor sets hardcoded defaults; `LoadPrefs()` overwrites from registry. `OnInitDialog()` calls neither directly (commented out at line 169); implies caller invokes `LoadPrefs()` before showing dialog.

## Data Flow Through This File

1. **Dialog Creation**: Parent constructs `CPrefsDlg`, calls `LoadPrefs()` to hydrate member variables from registry, then `DoModal()` to show modally.
2. **User Interaction**: MFC framework fires `DDX_*` callbacks on control change; member variables track live UI state.
3. **OnOK() (Save Path)**:
   - `UpdateData(TRUE)` pushes all control values → member variables
   - Copy slider values: `m_nMoveSpeed = m_wndCamSpeed.GetPos()`; compute derived: `m_nAngleSpeed = m_nMoveSpeed * 0.50`
   - `SavePrefs()` writes all members → registry
   - `Sys_UpdateWindows(W_ALL)` invalidates all radiant viewports (triggers repaint with new grid/snap settings)
   - `Undo_SetMaxSize()` reconfigures undo subsystem
   - `CDialog::OnOK()` closes dialog

4. **Registry Store**: ~80 preferences spanning rendering, UI layout, file paths, tool behavior, game-specific toggles.

## Learning Notes

- **Editor ≠ Engine**: Q3Radiant is a tool; preferences here do **not** affect runtime gameplay. The `.exe` paths and PAK files are for editor convenience (launching compiled maps, embedded asset browsing), not engine state.
- **Multi-game heritage**: Quake2/Quake3 toggle visible in `WHATGAME_KEY` branch logic reflects the original id Software toolkit's support for multiple game IWADs. Modern Q3 forks typically hardcode Quake3.
- **MFC idioms**: `GetProfileInt/String` uses `HKEY_CURRENT_USER\Software\<VendorName>\<AppName>\<Section>` under the hood; portable to UNIX via Wine/Proton, but fundamentally Windows-centric.
- **No persistence abstraction**: Preferences tightly coupled to registry; refactoring to JSON/INI would require wholesale replacement of ~50 `GetProfile*` calls.
- **Control-centric defaults**: Member variable defaults in constructor (lines 122–167) duplication `MOUSE_DEF`, `LOADLAST_DEF`, etc.; inconsistency risk if either set changes.

## Potential Issues

- **Uninitialized `m_nMoveSpeed` on first launch**: If `LoadPrefs()` never called or registry empty, `OnOK()` reads garbage `m_nMoveSpeed` and computes `m_nAngleSpeed = garbage * 0.50`. Constructor sets no value for `m_nMoveSpeed`/`m_nAngleSpeed`; only registry or `LoadPrefs()` initializes them. Default `MOVESPEED_KEY` is 400, but if registry is somehow wiped, undefined behavior.
- **Game-conditional UI not enforced**: Lines 261–275 disable PAK/BSP controls if Quake3 selected, but `SavePrefs()` still writes `m_strPAKFile` to registry regardless. Stale pak0.pak paths could mislead users.
- **`AddSlash()` dependency undeclared**: Line 227 calls `AddSlash(strPrefab)` with no `#include`; likely in a shared utility header not visible here.
