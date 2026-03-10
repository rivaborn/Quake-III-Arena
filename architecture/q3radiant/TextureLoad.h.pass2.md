# q3radiant/TextureLoad.h — Enhanced Analysis

## Architectural Role

`TextureLoad.h` defines a simple MFC dialog component used in the Q3Radiant level editor for texture selection and browsing. It sits entirely within the **editor subsystem** (q3radiant/) and has zero presence in the runtime engine. This is part of the map authoring toolchain, not the game itself—textures are already baked into BSP and referenced at compile-time by the map compiler (q3map/).

## Key Cross-References

### Incoming (who depends on this file)
- Called from the main editor window or brush/surface property dialogs when the user needs to select a texture from the available pool
- Instantiated as a modal dialog by parent UI windows in the editor
- No runtime engine dependencies; editor-only usage

### Outgoing (what this file depends on)
- MFC dialog framework (CDialog, CListBox, CDataExchange)
- Windows resource system (IDD_TEXLIST constant from resource.rc)
- Parent window context (CWnd* pParent) for modality
- The editor's texture asset system (implicitly populated into m_wndList at init time)

## Design Patterns & Rationale

**Standard MFC Modal Dialog Pattern:**
- Inherits from `CDialog`; uses `DoDataExchange` for data binding (DDX/DDV framework)
- `OnInitDialog` populates the list; `OnOK` commits the user's selection and closes
- The pattern is idiomatic to early-2000s Windows C++ — MFC dialogs were the standard UI approach before .NET

**Why a List Box?**
- Efficient scrolling for potentially large texture sets (Q3A ships with ~500+ unique textures)
- Simple selection model (single-select by default)
- No need for preview or filtering at this UI layer

## Data Flow Through This File

1. **Creation:** Parent window instantiates `CTextureLoad` and calls `DoModal()` (implicit MFC behavior)
2. **Initialization:** `OnInitDialog()` fires; texture asset system populates `m_wndList` with available texture names
3. **User Interaction:** User scrolls and clicks a texture name in the list
4. **Commit:** `OnOK()` is called; selected texture index/name is extracted and returned to parent via dialog result code
5. **Teardown:** Dialog closes; parent reads selection from the `CListBox`

## Learning Notes

- **Editor ≠ Engine Separation:** This file exemplifies the clear split: the editor is a Windows-only C++ tool; the runtime engine is cross-platform. Texture *selection* happens in the editor; the actual textures are compiled into the BSP by q3map/ or inlined in shaders by the renderer.
- **No Abstraction Over MFC:** Unlike modern engines (which separate UI from presentation), Q3Radiant directly uses MFC dialogs. This tightly couples the UI to Windows/VC++.
- **Resource Binding:** The `IDD_TEXLIST` enum ties this class to a `.rc` resource file (not shown here), which defines the dialog geometry, controls, and string resources. This is the MFC way of separating layout from code.
- **Texture System Unknown:** This header doesn't reveal *where* the texture list comes from—likely a static array or loaded from `baseq3/shaders/` at editor startup.

## Potential Issues

- No error handling in the visible interface (e.g., what if `m_wndList` is never populated?). Likely relies on parent to validate state.
- The `CListBox` is bare—no search, filtering, or preview. Users with 500+ textures must scroll manually.
- Windows-only; zero portability (MFC is not cross-platform).
