# q3radiant/TexEdit.cpp — Enhanced Analysis

## Architectural Role

`CTexEdit` is a lightweight MFC edit control wrapper for the Radiant level editor's texture browser. It provides real-time texture name filtering via keyboard input, bridging the UI input layer to the texture window's query pipeline. As part of the editor's visual asset management layer, it has no runtime engine dependencies—it exists entirely within the offline Radiant toolchain.

## Key Cross-References

### Incoming (who depends on this file)
- **TexWnd** (`q3radiant/TexWnd.cpp`) creates and manages the `CTexEdit` instance; owns the `m_pTexWnd` pointer that receives `UpdateFilter()` calls
- **MFC framework** (`stdafx.h`) provides the base `CEdit` class and message pump (`ON_WM_*` macros)

### Outgoing (what this file depends on)
- **TexWnd::UpdateFilter()** — called synchronously on every keystroke (`EN_CHANGE` event) to re-query/filter the texture list
- **MFC graphical subsystem** — `CDC`, `HBRUSH`, stock graphics objects (`LTGRAY_BRUSH`)
- **Win32 API** — `RGB()`, `GetStockObject()`, `SetBkColor()`, message reflection

## Design Patterns & Rationale

**Immediate-mode filtering**: Every keystroke triggers `UpdateFilter()` with the current text; the owning `TexWnd` performs a substring/regex match against loaded shader names. This is simple and responsive for Radiant's typical map-editing UI responsiveness (no debouncing or throttling needed).

**MFC message reflection** (`ON_CTLCOLOR_REFLECT`): The control requests custom rendering early in its message dispatch, allowing a child control to style itself without the parent needing to know implementation details.

**Lazy pointer initialization**: `m_pTexWnd = NULL` in constructor allows the control to exist before the window is constructed; the parent sets `m_pTexWnd` during initialization. This decouples construction order from lifecycle coupling.

**Custom font at creation time**: `OnCreate()` runs once per control instance, setting Arial 10pt as the texture-list editor font. This ensures consistent UI appearance across sessions without requiring external font resources.

## Data Flow Through This File

1. **User types character** → MFC delivers `WM_CHAR` → `CEdit` updates internal buffer
2. **Edit content changes** → `EN_CHANGE` notification reflected back to `CTexEdit`
3. **OnChange()** extracts buffer text via `GetWindowText()` and passes to `m_pTexWnd->UpdateFilter(str)`
4. **TexWnd** re-filters its internal shader/texture list and repaints the texture pane

The control itself stores no filtering state—it is purely a passthrough UI widget.

## Learning Notes

**Radiant is Windows/MFC-native**: Unlike the portable runtime engine (which uses abstraction layers like `GLimp_*` for graphics and `Sys_*` for I/O), the editor directly embeds MFC classes. This reflects 1990s-2000s game-tool conventions where editor code was rarely cross-platform.

**Separation from the engine**: The editor **never** directly interacts with runtime systems (renderer, client, game VM, botlib). Its texture/shader browsing is based on parsing static `.shader` files and the BSP's shader references—not the live rendering pipeline.

**Immediate UI feedback without threads**: The synchronous `UpdateFilter()` call in `EN_CHANGE` assumes `TexWnd::UpdateFilter()` is fast enough to not block the message loop. For Quake III's shader count (~200–500 per map), a linear substring search is acceptable.

## Potential Issues

None clearly inferable from the code. The implementation is straightforward. One minor observation: `m_pTexWnd` is never null-checked in `OnChange()`, so a use-before-initialization bug is theoretically possible if `UpdateFilter()` is called before the parent window sets the pointer—but MFC's message dispatch order makes this unlikely in practice.
