# q3radiant/TextureBar.cpp — Enhanced Analysis

## Architectural Role

CTextureBar is a dockable UI widget in the Radiant level editor (a separate Windows MFC application, not part of the runtime engine). It provides real-time texture coordinate manipulation (shift, scale, rotation) for selected brush faces during map editing. The widget maintains bidirectional synchronization between UI spin-controls and the underlying `texdef_t` structures of selected faces, dispatching live preview updates through editor selection handlers.

## Key Cross-References

### Incoming (who depends on this file)
- Radiant editor main window frame (`q3radiant/MainFrm.*`) creates and docks this dialog bar as a toolbar
- MFC message pump dispatches control notifications to `OnDeltapos*` handlers
- Selected face list (`g_ptrSelectedFaces`) is read/written by `GetSurfaceAttributes()` and `SetSurfaceAttributes()`

### Outgoing (what this file depends on)
- **Editor selection API**: `Select_ShiftTexture()`, `Select_ScaleTexture()`, `Select_RotateTexture()` — likely in `q3radiant/SELECT.*`
- **Editor globals**: 
  - `g_qeglobals.d_savedinfo.m_nTextureTweak` — tweak step size
  - `g_qeglobals.d_texturewin.texdef` — default texture definition
  - `g_ptrSelectedFaces` — array of pointers to selected `face_t` structures
- **Editor utilities**: `Sys_UpdateWindows(W_CAMERA)`, `Sys_Printf()` — viewport refresh and logging
- **Editor types**: `texdef_t`, `face_t` (BSP face structure with embedded texture definition)
- **MFC framework**: `CDialogBar`, `DDX_*` data exchange, `NM_UPDOWN` notifications

## Design Patterns & Rationale

- **MFC Message Map** (lines 74–86): Static dispatch table for spin-control notifications (`UDN_DELTAPOS`). Windows-idiomatic for the era; avoids virtual dispatch overhead.
- **Data Exchange (DDX)** (lines 59–71): `DoDataExchange()` marshals UI values ↔ member variables. MFC pattern that decouples UI state from code.
- **Synchronous Live Preview**: Each spin-control adjustment immediately calls a `Select_*Texture()` function and refreshes the 3D viewport (`Sys_UpdateWindows`), giving real-time feedback—critical for interactive editing.
- **Dual State Model**: Texture attributes come from either selected faces (`g_ptrSelectedFaces`) or the editor's default texture window, with selected faces taking precedence.

## Data Flow Through This File

1. **User Input → UI**: Spin control `iDelta` value captured in `OnDeltapos*` handlers
2. **UI → Selection API**: Magnitude + sign determine texture operation (e.g., `iDelta < 0` → positive shift)
3. **Selection API → World**: `Select_ShiftTexture()` modifies all selected faces in place
4. **World → Viewport**: `Sys_UpdateWindows(W_CAMERA)` triggers renderer refresh
5. **Apply Button** (`OnBtnApplytexturestuff`): `SetSurfaceAttributes()` commits UI spinbox values directly to selected face `texdef_t` structures
6. **Sync Back** (`GetSurfaceAttributes`): Reads first selected face's `texdef_t` into UI controls (after operations, for feedback)

## Learning Notes

- **Editor ≠ Engine**: This file belongs to the *offline* level editor (Radiant), not the `code/` runtime engine. The architecture context provided focuses on the runtime; this is a separate Windows MFC application with its own UI framework.
- **MFC Idioms**: The code exemplifies late-1990s Windows native UI patterns (message maps, DDX, `CDialogBar` docking). Modern engines use cross-platform frameworks (Qt, Dear ImGui, custom web-based UIs).
- **Synchronous Feedback Loop**: Texture edits apply instantly to the 3D view—no deferred batching. Reflects Radiant's immediate-mode philosophy for authoring UX.
- **Brush Primitive Placeholder**: The warning at `SetSurfaceAttributes()` (line 186) indicates that brush primitive texture mode (a later feature) was stubbed out but never completed in the editor; the runtime game supports it, but the editor UI does not.

## Potential Issues

- **Unimplemented Features**: Three TODO/warning sites:
  - Line 25: "the whole CTextureBar has to be modified for the new texture code" (suggests texture system refactoring never completed in the UI)
  - Line 156: `OnSelectionPrint()` empty—feature was planned but abandoned
  - Line 186: Brush primitive mode warning—safe warning, but indicates incomplete editor support for a runtime feature
- **No Bounds Checking**: `GetAt(0)` on `g_ptrSelectedFaces` assumes the list is non-empty; crash risk if called with no selection (though `GetSize() > 0` check guards most paths)
- **Single-Face Limit**: Only operates on the first selected face (`GetAt(0)`), even if multiple faces are selected; multi-face edits likely handled elsewhere or not supported
