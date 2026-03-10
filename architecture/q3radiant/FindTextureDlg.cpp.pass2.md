# q3radiant/FindTextureDlg.cpp — Enhanced Analysis

## Architectural Role

This file implements a modeless MFC dialog in Q3Radiant (the level editor) that provides a find-and-replace interface for textures across map geometry. It is **completely isolated from the runtime engine** — it exists only in the offline editing toolchain. The dialog manages ephemeral UI state (window position, focus, input fields) and delegates the actual texture replacement logic to editor-level functions (`FindReplaceTextures`), serving as a thin presentation layer for a common map-editing operation.

## Key Cross-References

### Incoming (who depends on this file)
- **Q3Radiant main UI** (`MainFrm.cpp` / `QE3.cpp`): Calls `CFindTextureDlg::show()` to display the dialog; may call `updateTextures(const char*)` to push selected texture names into the dialog fields
- **Editor texture selection workflows**: When a texture is selected in the texture browser, the dialog's `updateTextures()` hook fires to populate either Find or Replace field depending on which field has focus

### Outgoing (what this file depends on)
- **`FindReplaceTextures()`** (undefined in cross-reference; must be in editor core): Performs the actual find-and-replace on all selected faces or entire map
- **`SaveRegistryInfo()` / `LoadRegistryInfo()`** (editor utility functions): Persists window position/size to Windows registry under key `"Radiant::TextureFindWindow"`
- **Windows/MFC framework**: Dialog creation, message routing, data exchange (`DDX_Check`, `DDX_Text`)

## Design Patterns & Rationale

1. **Singleton with global reference**: `g_TexFindDlg` (true singleton) and `g_dlgFind` (alias reference) allow any UI component to invoke `show()` or `updateTextures()` without passing a dialog pointer. This is typical of 1990s–2000s MFC editor UIs where global state was acceptable.

2. **Modeless dialog lifecycle**: Unlike modal dialogs (which block execution), this dialog can remain open while the user edits. The code defensively checks `GetSafeHwnd()` and `IsWindow()` to handle the case where the user closed the window without destroying it.

3. **Live-update dispatch pattern**: The `g_bFindActive` static flag tracks which field (Find vs. Replace) had focus, so when `updateTextures(const char *p)` is called externally (e.g., user clicks a texture), the field is automatically populated. The `m_bLive` checkbox allows the user to disable this auto-population.

4. **Registry persistence**: Window position is saved on Apply/OK/Cancel, restored on `show()`. This preserves editor state across sessions—a common 1990s UI convention.

5. **MFC message map + DDX data exchange**: Separates UI layout (defined in `.rc` resource file, not shown) from logic. The `DoDataExchange` method marshals data between dialog controls and member variables (`m_strFind`, `m_strReplace`, `m_bSelectedOnly`, `m_bForce`, `m_bLive`).

## Data Flow Through This File

```
External event (texture selected)
  → updateTextures(const char *p)
    → Check if dialog isOpen() && m_bLive enabled
      → Call setFindStr() or setReplaceStr() based on g_bFindActive
        → UpdateData(FALSE) pushes member var to UI control

User clicks Apply/OK
  → OnBtnApply() / OnOK()
    → UpdateData(TRUE) pulls dialog state into members
    → SaveRegistryInfo() (window persistence)
    → FindReplaceTextures(m_strFind, m_strReplace, m_bSelectedOnly, m_bForce)
      (external function applies changes to map)
```

The dialog itself does **not** modify the map; it is purely a wrapper around user input capture and state presentation.

## Learning Notes

1. **MFC-era UI patterns**: This code exemplifies pre-.NET Windows UI development. Modern C++ editors (e.g., VS Code, Jetbrains IDEs) would implement this in HTML/CSS/JS or Qt, avoiding Windows-specific framework lock-in. Q3Radiant was Windows-only.

2. **Global state as a feature**: The singleton pattern with global references is considered an anti-pattern in modern OOP, but in 1999–2005 editor codebases it was pragmatic: it avoided callback chains and made the dialog accessible from any UI context without dependency injection.

3. **Registry for UI persistence**: Saving window geometry to the Windows registry is idiomatic for native Windows apps. Modern cross-platform editors use JSON or YAML config files in `~/.config` or `%APPDATA%`.

4. **Live-update toggle** (`m_bLive`): This is a usability feature—some users want automatic population of the Find field when clicking textures; others find it intrusive. The checkbox allows user preference.

5. **No engine involvement**: Unlike runtime code, this dialog never touches collision, rendering, entity systems, or game simulation. It is **purely offline editor tooling**, completely decoupled from the engine architecture described in the codebase overview.

## Potential Issues

None clearly identifiable from code structure alone, though:
- The global `g_bFindActive` flag assumes single-threaded execution (likely safe in a 1990s editor, but brittle if ever parallelized).
- No visible error handling if `FindReplaceTextures()` fails; the user receives no feedback.
- Registry persistence is Windows-only; a cross-platform port would need refactoring.
