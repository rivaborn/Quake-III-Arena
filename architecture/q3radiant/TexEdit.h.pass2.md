# q3radiant/TexEdit.h — Enhanced Analysis

## Architectural Role

`CTexEdit` is a thin MFC wrapper around the Windows `CEdit` control, specializing it for inline texture/shader parameter editing within the Q3Radiant level editor's texture browser UI. It bridges the editor's texture window (`CTexWnd`) with the native text-input control, applying custom font rendering and responding to user edits. This component is part of the editor's **texture properties/attributes panel**, enabling rapid iteration on shader parameters during map construction.

## Key Cross-References

### Incoming (who depends on this file)
- `TexWnd.h`/`TexWnd.cpp` — Creates and owns `CTexEdit` instances as child controls within the texture viewing/browsing window
- `TextureBar.h`/`TextureBar.cpp` — Likely hosts the texture window and manages the overall texture UI layout
- `MainFrm.h`/`MainFrm.cpp` — Top-level editor frame that contains the texture bar/docking panels

### Outgoing (what this file depends on)
- MFC framework: `CEdit` (base class), `CFont` (member), message dispatch macros
- `CTexWnd` — Parent window pointer used for callbacks on `OnChange` events
- CDC (device context) — Passed to `CtlColor` for custom color rendering
- Windows GDI — Indirect dependency via MFC for font and color handling

## Design Patterns & Rationale

**MFC Message-Driven Architecture**: The `//{{AFX_*}}` comments denote MFC ClassWizard regions (early 2000s code generation). The design reflects a **visual control hierarchy** pattern:
- Derive from standard control (`CEdit`)
- Override message handlers to customize behavior
- Maintain back-pointer to parent (`m_pTexWnd`) for event notification

**Why this structure?** MFC controls in the late 1990s-early 2000s were the primary UI toolkit for professional Windows tools. Deriving a custom control class allowed designers to add editor-specific behavior (custom fonts, live texture preview updates) without reimplementing the entire text-input machinery.

**What's notably *absent***: No data validation, no undo/redo integration visible here, and no separation between view and model. By modern standards, this is tightly coupled to the rendering context.

## Data Flow Through This File

1. **Creation**: `OnCreate()` initializes font styling (likely monospace for shader parameters)
2. **User Input**: User types shader/material parameters; MFC invokes `OnChange()`
3. **Parent Notification**: `OnChange()` likely calls back into `m_pTexWnd` to trigger preview/validation
4. **Rendering**: `CtlColor()` customizes text/background color (possibly syntax highlighting or error indication)
5. **Data Out**: Parent window reads the edit control's text content via standard `GetWindowText()` or similar

## Learning Notes

**Idiomatic to this era & engine**:
- **MFC "message cracking"** was the standard UI pattern for C++ Windows tools in the Radiant/Q3A era (1999-2005). Modern engines use event-driven systems (Qt, Electron, custom frameworks).
- **Back-pointers from controls to parents** (e.g., `m_pTexWnd`) were common; modern UI frameworks use signals/slots, observables, or binding layers.
- **GDI font management** via `CFont` was manual; contemporary editors use font atlases and GPU-resident glyph rendering.

**Game engine connection**: While the runtime engine (`code/renderer/`, `code/game/`) uses a data-driven shader system and deferred rendering, the **editor** embeds shader editing UI directly in the tool, reflecting the iterative nature of content creation pipelines of that era. Modern engines (Unreal, Unity) separate shader editing into specialized systems or external plugins.

## Potential Issues

- **No bounds/validation**: The control accepts any text without checking syntax, meaning invalid shader parameters can be entered and sent to the parent window. Parent must handle validation.
- **Tight coupling**: The back-pointer to `CTexWnd` creates a hard dependency; if `CTexWnd` outlives or is freed before `CTexEdit`, a dangling pointer results.
- **MFC ClassWizard artifacts**: The `//{{AFX_*}}` markers indicate the file is partially code-generated and may not integrate well with modern build systems or version control (these regions can be fragile).
- **No threading safety**: If the editor is multi-threaded (unlikely for a 2005 tool), the callback to `m_pTexWnd` from `OnChange` could race; however, MFC is single-threaded-UI by design.
